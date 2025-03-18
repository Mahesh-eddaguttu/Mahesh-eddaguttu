const express = require('express');
const cors = require('cors');
const axios = require('axios');
const sqlite3 = require('sqlite3').verbose();

const app = express();
app.use(cors());
app.use(express.json());

const db = new sqlite3.Database('./data.db', (err) => {
  if (err) console.error('Failed to connect to database:', err.message);
  else console.log('Connected to SQLite database');
});

// Create tables for products and orders
db.serialize(() => {
  db.run("CREATE TABLE IF NOT EXISTS products (id INTEGER PRIMARY KEY, title TEXT, url TEXT)");
  db.run("CREATE TABLE IF NOT EXISTS orders (id INTEGER PRIMARY KEY, order_data TEXT)");
});

let storeUrl = '';
let apiKey = '';
const MAX_RETRIES = 3;

// Retry function with exponential backoff
const retryRequest = async (fn, retries = MAX_RETRIES, delay = 1000) => {
  try {
    return await fn();
  } catch (err) {
    if (retries <= 0) throw err;
    console.log(`Retrying... attempts left: ${retries}`);
    await new Promise((resolve) => setTimeout(resolve, delay));
    return retryRequest(fn, retries - 1, delay * 2);
  }
};

app.post('/connect-store', (req, res) => {
  const { storeUrl: url, apiKey: key } = req.body;
  if (!url || !key) {
    return res.status(400).json({ message: 'Store URL and API Key are required' });
  }
  storeUrl = url;
  apiKey = key;
  res.json({ message: 'Store connected successfully' });
});

app.post('/import-product', async (req, res) => {
  const { productUrl } = req.body;
  try {
    const productData = await retryRequest(() => axios.get(productUrl));
    const title = productData.data.title;

    db.run('INSERT INTO products (title, url) VALUES (?, ?)', [title, productUrl], (err) => {
      if (err) return res.status(500).json({ message: 'Failed to save product' });
      res.json({ product: `Product imported: ${title}` });
    });
  } catch (error) {
    console.error('Failed after retries:', error.message);
    res.status(500).json({ message: 'Failed to import product after retries' });
  }
});

app.get('/get-products', async (req, res) => {
  try {
    const response = await retryRequest(() => axios.get(`${storeUrl}/products.json`, {
      headers: { 'X-API-KEY': apiKey }
    }));
    const products = response.data.products;

    products.forEach((p) => {
      db.run('INSERT OR IGNORE INTO products (title, url) VALUES (?, ?)', [p.title, p.url]);
    });

    res.json({ products });
  } catch (error) {
    console.error('Failed after retries:', error.message);
    db.all('SELECT * FROM products', (err, rows) => {
      if (err) return res.status(500).json({ message: 'Failed to fetch local products' });
      res.json({ products: rows });
    });
  }
});

app.get('/get-orders', async (req, res) => {
  try {
    const response = await retryRequest(() => axios.get(`${storeUrl}/orders.json`, {
      headers: { 'X-API-KEY': apiKey }
    }));
    const orders = response.data.orders;

    orders.forEach((o) => {
      db.run('INSERT OR IGNORE INTO orders (order_data) VALUES (?)', [JSON.stringify(o)]);
    });

    res.json({ orders });
  } catch (error) {
    console.error('Failed after retries:', error.message);
    db.all('SELECT * FROM orders', (err, rows) => {
      if (err) return res.status(500).json({ message: 'Failed to fetch local orders' });
      res.json({ orders: rows.map(row => JSON.parse(row.order_data)) });
    });
  }
});

app.listen(5000, () => console.log('Server running on http://localhost:5000'));

