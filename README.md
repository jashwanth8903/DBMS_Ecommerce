# DBMS Project: E-Commerce Management System

## 📌 Mini World Description

This project simulates an e-commerce platform that manages customers, sellers, products, carts, and payments. Customers can browse and add products to their cart, make payments, and manage their profiles. Sellers can add products, and administrators can monitor overall activity and revenue.

---

## 📚 Project Structure

### ✅ Functional Requirements

* Customer registration and authentication
* Seller registration and product management
* Adding items to the cart
* Making payments
* Order history and filtering
* Admin features for monitoring activity

### 🧠 Entity-Relation (ER) Diagram

Entities:

* Customer
* Seller
* Product
* Cart
* Cart Item
* Payment
* Seller Phone Number

Relationships:

* Customer owns Cart
* Cart contains Cart Items
* Cart Items link to Products
* Products belong to Sellers
* Customers make Payments
* Sellers have multiple Phone Numbers

Constraints:

* On delete SET NULL or CASCADE appropriately used

---

## 🗃️ Relational Schema & SQL Implementation

### 3.1 Creating Tables

```sql
CREATE TABLE Cart (
  Cart_id VARCHAR(7) NOT NULL,
  PRIMARY KEY(Cart_id)
);

CREATE TABLE Customer (
  Customer_id VARCHAR(6) NOT NULL,
  c_pass VARCHAR(10) NOT NULL,
  Name VARCHAR(20) NOT NULL,
  Address VARCHAR(20) NOT NULL,
  Pincode NUMBER(6) NOT NULL,
  Phone_number_s NUMBER(10) NOT NULL,
  Cart_id VARCHAR(7) NOT NULL,
  PRIMARY KEY (Customer_id),
  FOREIGN KEY(Cart_id) REFERENCES Cart(Cart_id)
);

CREATE TABLE Seller (
  Seller_id VARCHAR(6) NOT NULL,
  s_pass VARCHAR(10) NOT NULL,
  Name VARCHAR(20) NOT NULL,
  Address VARCHAR(10) NOT NULL,
  PRIMARY KEY (Seller_id)
);

CREATE TABLE Seller_Phone_num (
  Phone_num NUMBER(10) NOT NULL,
  Seller_id VARCHAR(6) NOT NULL,
  PRIMARY KEY (Phone_num, Seller_id),
  FOREIGN KEY (Seller_id) REFERENCES Seller(Seller_id) ON DELETE CASCADE
);

CREATE TABLE Payment (
  payment_id VARCHAR(7) NOT NULL,
  payment_date DATE NOT NULL,
  Payment_type VARCHAR(10) NOT NULL,
  Customer_id VARCHAR(6) NOT NULL,
  Cart_id VARCHAR(7) NOT NULL,
  total_amount NUMERIC(6),
  PRIMARY KEY (payment_id),
  FOREIGN KEY (Customer_id) REFERENCES Customer(Customer_id),
  FOREIGN KEY (Cart_id) REFERENCES Cart(Cart_id)
);

CREATE TABLE Product (
  Product_id VARCHAR(7) NOT NULL,
  Type VARCHAR(7) NOT NULL,
  Color VARCHAR(15) NOT NULL,
  P_Size VARCHAR(2) NOT NULL,
  Gender CHAR(1) NOT NULL,
  Commission NUMBER(2) NOT NULL,
  Cost NUMBER(5) NOT NULL,
  Quantity NUMBER(2) NOT NULL,
  Seller_id VARCHAR(6),
  PRIMARY KEY (Product_id),
  FOREIGN KEY (Seller_id) REFERENCES Seller(Seller_id) ON DELETE SET NULL
);

CREATE TABLE Cart_item (
  Quantity_wished NUMBER(1) NOT NULL,
  Date_Added DATE NOT NULL,
  Cart_id VARCHAR(7) NOT NULL,
  Product_id VARCHAR(7) NOT NULL,
  purchased VARCHAR(3) DEFAULT 'NO',
  PRIMARY KEY(Cart_id, Product_id),
  FOREIGN KEY (Cart_id) REFERENCES Cart(Cart_id),
  FOREIGN KEY (Product_id) REFERENCES Product(Product_id)
);
```

### 3.2 Inserting Sample Data

```sql
INSERT INTO Cart VALUES('crt1011');

INSERT INTO Customer VALUES('cid100','ABCM1235','rajat','G-453',632014,9893135876, 'crt1011');

INSERT INTO Seller VALUES('sid100','12345','aman','delhi cmc');

INSERT INTO Product VALUES('pid1001','jeans','red','32','M',10,10005,20,'sid100');

INSERT INTO Seller_Phone_num VALUES(9943336206,'sid100');

INSERT INTO Cart_item VALUES(3, TO_DATE('10-OCT-1999','dd-mon-yyyy'),'crt1011','pid1001','Y');

INSERT INTO Payment VALUES('pmt1001', TO_DATE('10-OCT-1999','dd-mon-yyyy'),'online','cid100','crt1011', NULL);
```

---

## 🔍 Queries

### 4.1 Basic Queries

**1. View products in the cart**

```sql
SELECT * FROM product
WHERE product_id IN (
  SELECT product_id FROM Cart_item
  WHERE Cart_id IN (SELECT Cart_id FROM Customer WHERE Customer_id='cid100') AND purchased='NO'
);
```

**2. View order history**

```sql
SELECT product_id, Quantity_wished FROM Cart_item
WHERE purchased='Y' AND Cart_id IN (SELECT Cart_id FROM Customer WHERE Customer_id='cid101');
```

**3. Filtered products**

```sql
SELECT product_id, color, cost, seller_id FROM product
WHERE type='jeans' AND p_size='32' AND gender='F' AND quantity>0;
```

**4. Modify cart**

```sql
DELETE FROM cart_item
WHERE product_id='pid1001' AND Cart_id IN (SELECT Cart_id FROM Customer WHERE Customer_id='cid100');
```

**5. Seller stops selling**

```sql
DELETE FROM seller WHERE seller_id = 'sid100';
UPDATE product SET quantity = 0 WHERE seller_id IS NULL;
```

**6. Products purchased on a particular date**

```sql
SELECT product_id FROM cart_item
WHERE purchased='Y' AND date_added=TO_DATE('12-DEC-2018', 'dd-mon-yyyy');
```

**7. Count of products sold on each date**

```sql
SELECT COUNT(product_id) AS count_pid, date_added FROM Cart_item
WHERE purchased='Y'
GROUP BY date_added;
```

**8. Total cart price for a customer**

```sql
SELECT SUM(quantity_wished * cost) AS total_payable
FROM product p
JOIN cart_item c ON p.product_id = c.product_id
WHERE c.cart_id IN (SELECT Cart_id FROM customer WHERE customer_id='cid101') AND purchased='Y';
```

**9. Customers who have not purchased anything**

```sql
SELECT * FROM customer
WHERE customer_id NOT IN (SELECT customer_id FROM Payment);
```

**10. Website profit from sales**

```sql
SELECT SUM(quantity_wished * cost * commission / 100) AS total_profit
FROM product p
JOIN cart_item c ON p.product_id = c.product_id
WHERE purchased='Y';
```

---


## 💡 Extra Features (Implemented with SQL Code)

### 1. Trigger: Auto-update inventory after purchase

```sql
CREATE OR REPLACE TRIGGER update_inventory
AFTER INSERT ON cart_item
FOR EACH ROW
WHEN (NEW.purchased = 'Y')
BEGIN
  UPDATE product
  SET quantity = quantity - :NEW.quantity_wished
  WHERE product_id = :NEW.product_id;
END;
/
```

### 2. Function: Return most popular product (based on quantity sold)

```sql
CREATE OR REPLACE FUNCTION get_most_popular_product RETURN VARCHAR2 IS
  popular_product VARCHAR2(20);
BEGIN
  SELECT product_id INTO popular_product
  FROM (
    SELECT product_id, SUM(quantity_wished) AS total_sold
    FROM cart_item
    WHERE purchased = 'Y'
    GROUP BY product_id
    ORDER BY total_sold DESC
  ) WHERE ROWNUM = 1;
  RETURN popular_product;
END;
/
```

### 3. Stored Procedure: Checkout process

```sql
CREATE OR REPLACE PROCEDURE checkout_cart (
  p_customer_id IN VARCHAR2,
  p_cart_id IN VARCHAR2,
  p_payment_type IN VARCHAR2
) AS
  v_total NUMBER;
  v_payment_id VARCHAR2(10);
BEGIN
  SELECT SUM(c.quantity_wished * p.cost)
  INTO v_total
  FROM cart_item c JOIN product p
  ON c.product_id = p.product_id
  WHERE c.cart_id = p_cart_id AND c.purchased = 'N';

  SELECT 'pmt' || TO_CHAR(SYSDATE, 'HH24MISS') INTO v_payment_id FROM dual;

  INSERT INTO payment VALUES(v_payment_id, SYSDATE, p_payment_type, p_customer_id, p_cart_id, v_total);

  UPDATE cart_item SET purchased = 'Y'
  WHERE cart_id = p_cart_id AND purchased = 'N';
END;
/
```

### 4. View: Admin dashboard to show total revenue by date

```sql
CREATE OR REPLACE VIEW admin_dashboard AS
SELECT p.payment_date, SUM(p.total_amount) AS total_revenue
FROM payment p
GROUP BY p.payment_date;
```

---

## 📎 How to Run

1. Use MySQL or any other compatible RDBMS
2. Create tables in the provided order
3. Insert sample data
4. Execute queries to verify logic

---

## 📌 Author

**Project by:** K. Jashwanth