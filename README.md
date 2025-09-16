# -assignment-week-8--- ==============================================

-- E-COMMERCE DATABASE MANAGEMENT SYSTEM
-- ==============================================
-- Author: [Your Name]
-- Date: [Current Date]
-- Description: Complete e-commerce database with customers, products, orders, payments, and inventory
-- ==============================================

-- Create database
CREATE DATABASE IF NOT EXISTS ecommerce_db;
USE ecommerce_db;

-- ==============================================
-- TABLE 1: Customers
-- ==============================================
CREATE TABLE customers (
    customer_id INT AUTO_INCREMENT PRIMARY KEY,
    first_name VARCHAR(50) NOT NULL,
    last_name VARCHAR(50) NOT NULL,
    email VARCHAR(100) UNIQUE NOT NULL,
    password_hash VARCHAR(255) NOT NULL,
    phone VARCHAR(20),
    date_of_birth DATE,
    registration_date TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    last_login TIMESTAMP NULL,
    is_active BOOLEAN DEFAULT TRUE,

    -- Constraints
    CONSTRAINT chk_email CHECK (email LIKE '%@%.%'),
    CONSTRAINT chk_dob CHECK (date_of_birth <= DATE_SUB(CURRENT_DATE, INTERVAL 13 YEAR)),
    
    -- Indexes
    INDEX idx_customer_email (email),
    INDEX idx_customer_name (last_name, first_name)
);

-- ==============================================
-- TABLE 2: Addresses (One-to-Many with Customers)
-- ==============================================
CREATE TABLE addresses (
    address_id INT AUTO_INCREMENT PRIMARY KEY,
    customer_id INT NOT NULL,
    address_type ENUM('billing', 'shipping') NOT NULL,
    street_address VARCHAR(255) NOT NULL,
    city VARCHAR(100) NOT NULL,
    state VARCHAR(100),
    country VARCHAR(100) NOT NULL,
    postal_code VARCHAR(20) NOT NULL,
    is_default BOOLEAN DEFAULT FALSE,

    -- Foreign key
    FOREIGN KEY (customer_id) REFERENCES customers(customer_id) ON DELETE CASCADE,
    
    -- Constraints
    CONSTRAINT unique_default_address UNIQUE (customer_id, address_type, is_default),
    
    -- Indexes
    INDEX idx_customer_address (customer_id, address_type)
);

-- ==============================================
-- TABLE 3: Categories
-- ==============================================
CREATE TABLE categories (
    category_id INT AUTO_INCREMENT PRIMARY KEY,
    category_name VARCHAR(100) NOT NULL UNIQUE,
    description TEXT,
    parent_category_id INT NULL,

    -- Self-referencing foreign key for subcategories
    FOREIGN KEY (parent_category_id) REFERENCES categories(category_id) ON DELETE SET NULL,
    
    -- Indexes
    INDEX idx_category_parent (parent_category_id)
);

-- ==============================================
-- TABLE 4: Products
-- ==============================================
CREATE TABLE products (
    product_id INT AUTO_INCREMENT PRIMARY KEY,
    product_name VARCHAR(255) NOT NULL,
    description TEXT,
    category_id INT NOT NULL,
    price DECIMAL(10, 2) NOT NULL CHECK (price >= 0),
    cost_price DECIMAL(10, 2) CHECK (cost_price >= 0),
    sku VARCHAR(100) UNIQUE NOT NULL,
    weight_kg DECIMAL(8, 3) CHECK (weight_kg >= 0),
    is_active BOOLEAN DEFAULT TRUE,
    created_date TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    last_updated TIMESTAMP DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,

    -- Foreign key
    FOREIGN KEY (category_id) REFERENCES categories(category_id) ON DELETE RESTRICT,
    
    -- Indexes
    INDEX idx_product_category (category_id),
    INDEX idx_product_sku (sku),
    INDEX idx_product_price (price)
);

-- ==============================================
-- TABLE 5: Product Inventory
-- ==============================================
CREATE TABLE product_inventory (
    inventory_id INT AUTO_INCREMENT PRIMARY KEY,
    product_id INT NOT NULL UNIQUE,
    quantity_in_stock INT NOT NULL DEFAULT 0 CHECK (quantity_in_stock >= 0),
    low_stock_threshold INT DEFAULT 5,
    last_restocked DATE,

    -- Foreign key
    FOREIGN KEY (product_id) REFERENCES products(product_id) ON DELETE CASCADE,
    
    -- Indexes
    INDEX idx_inventory_stock (quantity_in_stock)
);

-- ==============================================
-- TABLE 6: Product Images (One-to-Many with Products)
-- ==============================================
CREATE TABLE product_images (
    image_id INT AUTO_INCREMENT PRIMARY KEY,
    product_id INT NOT NULL,
    image_url VARCHAR(500) NOT NULL,
    alt_text VARCHAR(255),
    is_primary BOOLEAN DEFAULT FALSE,
    display_order INT DEFAULT 0,

    -- Foreign key
    FOREIGN KEY (product_id) REFERENCES products(product_id) ON DELETE CASCADE,
    
    -- Constraints
    CONSTRAINT unique_primary_image UNIQUE (product_id, is_primary),
    
    -- Indexes
    INDEX idx_product_images (product_id, display_order)
);

-- ==============================================
-- TABLE 7: Orders
-- ==============================================
CREATE TABLE orders (
    order_id INT AUTO_INCREMENT PRIMARY KEY,
    customer_id INT NOT NULL,
    order_date TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    status ENUM('pending', 'confirmed', 'processing', 'shipped', 'delivered', 'cancelled') DEFAULT 'pending',
    total_amount DECIMAL(12, 2) NOT NULL CHECK (total_amount >= 0),
    shipping_address_id INT NOT NULL,
    billing_address_id INT NOT NULL,
    shipping_cost DECIMAL(8, 2) DEFAULT 0 CHECK (shipping_cost >= 0),
    tax_amount DECIMAL(8, 2) DEFAULT 0 CHECK (tax_amount >= 0),
    notes TEXT,

    -- Foreign keys
    FOREIGN KEY (customer_id) REFERENCES customers(customer_id) ON DELETE RESTRICT,
    FOREIGN KEY (shipping_address_id) REFERENCES addresses(address_id),
    FOREIGN KEY (billing_address_id) REFERENCES addresses(address_id),
    
    -- Indexes
    INDEX idx_order_customer (customer_id),
    INDEX idx_order_date (order_date),
    INDEX idx_order_status (status)
);

-- ==============================================
-- TABLE 8: Order Items (Many-to-Many between Orders and Products)
-- ==============================================
CREATE TABLE order_items (
    order_item_id INT AUTO_INCREMENT PRIMARY KEY,
    order_id INT NOT NULL,
    product_id INT NOT NULL,
    quantity INT NOT NULL CHECK (quantity > 0),
    unit_price DECIMAL(10, 2) NOT NULL CHECK (unit_price >= 0),
    discount DECIMAL(8, 2) DEFAULT 0 CHECK (discount >= 0),
    line_total DECIMAL(10, 2) GENERATED ALWAYS AS (quantity * (unit_price - discount)) STORED,

    -- Foreign keys
    FOREIGN KEY (order_id) REFERENCES orders(order_id) ON DELETE CASCADE,
    FOREIGN KEY (product_id) REFERENCES products(product_id) ON DELETE RESTRICT,
    
    -- Constraints
    CONSTRAINT unique_order_product UNIQUE (order_id, product_id),
    
    -- Indexes
    INDEX idx_order_items_order (order_id),
    INDEX idx_order_items_product (product_id)
);

-- ==============================================
-- TABLE 9: Payments
-- ==============================================
CREATE TABLE payments (
    payment_id INT AUTO_INCREMENT PRIMARY KEY,
    order_id INT NOT NULL,
    payment_method ENUM('credit_card', 'debit_card', 'paypal', 'bank_transfer') NOT NULL,
    payment_status ENUM('pending', 'completed', 'failed', 'refunded') DEFAULT 'pending',
    amount DECIMAL(10, 2) NOT NULL CHECK (amount >= 0),
    transaction_id VARCHAR(255),
    payment_date TIMESTAMP NULL,
    created_date TIMESTAMP DEFAULT CURRENT_TIMESTAMP,

    -- Foreign key
    FOREIGN KEY (order_id) REFERENCES orders(order_id) ON DELETE CASCADE,
    
    -- Constraints
    CONSTRAINT unique_transaction UNIQUE (transaction_id),
    
    -- Indexes
    INDEX idx_payment_order (order_id),
    INDEX idx_payment_status (payment_status),
    INDEX idx_payment_date (payment_date)
);

-- ==============================================
-- TABLE 10: Reviews (Many-to-Many between Customers and Products)
-- ==============================================
CREATE TABLE reviews (
    review_id INT AUTO_INCREMENT PRIMARY KEY,
    customer_id INT NOT NULL,
    product_id INT NOT NULL,
    rating INT NOT NULL CHECK (rating BETWEEN 1 AND 5),
    review_text TEXT,
    review_date TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    is_approved BOOLEAN DEFAULT FALSE,

    -- Foreign keys
    FOREIGN KEY (customer_id) REFERENCES customers(customer_id) ON DELETE CASCADE,
    FOREIGN KEY (product_id) REFERENCES products(product_id) ON DELETE CASCADE,
    
    -- Constraints
    CONSTRAINT unique_customer_product_review UNIQUE (customer_id, product_id),
    
    -- Indexes
    INDEX idx_review_product (product_id),
    INDEX idx_review_rating (rating),
    INDEX idx_review_date (review_date)
);

-- ==============================================
-- TABLE 11: Wishlists (Many-to-Many between Customers and Products)
-- ==============================================
CREATE TABLE wishlists (
    wishlist_id INT AUTO_INCREMENT PRIMARY KEY,
    customer_id INT NOT NULL,
    product_id INT NOT NULL,
    added_date TIMESTAMP DEFAULT CURRENT_TIMESTAMP,

    -- Foreign keys
    FOREIGN KEY (customer_id) REFERENCES customers(customer_id) ON DELETE CASCADE,
    FOREIGN KEY (product_id) REFERENCES products(product_id) ON DELETE CASCADE,
    
    -- Constraints
    CONSTRAINT unique_wishlist_item UNIQUE (customer_id, product_id),
    
    -- Indexes
    INDEX idx_wishlist_customer (customer_id),
    INDEX idx_wishlist_product (product_id)
);

-- ==============================================
-- TABLE 12: Coupons/Discounts
-- ==============================================
CREATE TABLE coupons (
    coupon_id INT AUTO_INCREMENT PRIMARY KEY,
    coupon_code VARCHAR(50) UNIQUE NOT NULL,
    description TEXT,
    discount_type ENUM('percentage', 'fixed') NOT NULL,
    discount_value DECIMAL(8, 2) NOT NULL CHECK (discount_value >= 0),
    min_order_amount DECIMAL(10, 2) DEFAULT 0,
    max_discount_amount DECIMAL(10, 2),
    start_date DATE NOT NULL,
    end_date DATE NOT NULL,
    usage_limit INT DEFAULT NULL,
    times_used INT DEFAULT 0,
    is_active BOOLEAN DEFAULT TRUE,

    -- Constraints
    CONSTRAINT chk_dates CHECK (start_date <= end_date),
    
    -- Indexes
    INDEX idx_coupon_code (coupon_code),
    INDEX idx_coupon_dates (start_date, end_date)
);

-- ==============================================
-- TABLE 13: Order Coupons (Many-to-Many between Orders and Coupons)
-- ==============================================
CREATE TABLE order_coupons (
    order_id INT NOT NULL,
    coupon_id INT NOT NULL,
    discount_applied DECIMAL(8, 2) NOT NULL CHECK (discount_applied >= 0),

    -- Composite primary key
    PRIMARY KEY (order_id, coupon_id),
    
    -- Foreign keys
    FOREIGN KEY (order_id) REFERENCES orders(order_id) ON DELETE CASCADE,
    FOREIGN KEY (coupon_id) REFERENCES coupons(coupon_id) ON DELETE RESTRICT
);

-- ==============================================
-- TABLE 14: Shipping Methods
-- ==============================================
CREATE TABLE shipping_methods (
    method_id INT AUTO_INCREMENT PRIMARY KEY,
    method_name VARCHAR(100) NOT NULL,
    description TEXT,
    base_cost DECIMAL(8, 2) NOT NULL CHECK (base_cost >= 0),
    cost_per_kg DECIMAL(8, 2) DEFAULT 0,
    estimated_days_min INT NOT NULL,
    estimated_days_max INT NOT NULL,
    is_active BOOLEAN DEFAULT TRUE,

    -- Constraints
    CONSTRAINT chk_estimated_days CHECK (estimated_days_min <= estimated_days_max),
    
    -- Indexes
    INDEX idx_shipping_method (method_name)
);

-- ==============================================
-- TABLE 15: Order Shipping (One-to-One with Orders)
-- ==============================================
CREATE TABLE order_shipping (
    order_id INT PRIMARY KEY,
    shipping_method_id INT NOT NULL,
    tracking_number VARCHAR(255) UNIQUE,
    shipped_date TIMESTAMP NULL,
    estimated_delivery DATE,
    actual_delivery DATE,

    -- Foreign keys
    FOREIGN KEY (order_id) REFERENCES orders(order_id) ON DELETE CASCADE,
    FOREIGN KEY (shipping_method_id) REFERENCES shipping_methods(method_id) ON DELETE RESTRICT,
    
    -- Indexes
    INDEX idx_tracking (tracking_number),
    INDEX idx_delivery_dates (estimated_delivery, actual_delivery)
);

-- ==============================================
-- INSERT SAMPLE DATA
-- ==============================================

-- Insert sample categories
INSERT INTO categories (category_name, description) VALUES
('Electronics', 'Electronic devices and accessories'),
('Clothing', 'Fashion and apparel'),
('Books', 'Books and literature'),
('Home & Garden', 'Home improvement and gardening supplies');

-- Insert sample products
INSERT INTO products (product_name, description, category_id, price, cost_price, sku, weight_kg) VALUES
('Smartphone X', 'Latest smartphone with advanced features', 1, 899.99, 600.00, 'ELEC-SMART-X', 0.2),
('Wireless Headphones', 'Noise-cancelling wireless headphones', 1, 199.99, 120.00, 'ELEC-HEAD-WL', 0.3),
('Cotton T-Shirt', '100% cotton comfortable t-shirt', 2, 29.99, 15.00, 'CLOTH-TS-COT', 0.1),
('Programming Book', 'Complete guide to programming', 3, 49.99, 25.00, 'BOOK-PROG-001', 0.4);

-- Insert sample inventory
INSERT INTO product_inventory (product_id, quantity_in_stock, low_stock_threshold, last_restocked) VALUES
(1, 50, 10, '2024-01-15'),
(2, 25, 5, '2024-01-10'),
(3, 100, 20, '2024-01-05'),
(4, 30, 5, '2024-01-12');

-- Insert sample customers
INSERT INTO customers (first_name, last_name, email, password_hash, phone) VALUES
('John', 'Doe', '<john.doe@email.com>', 'hashed_password_123', '555-0101'),
('Jane', 'Smith', '<jane.smith@email.com>', 'hashed_password_456', '555-0102');

-- ==============================================
-- CREATE VIEWS FOR REPORTING
-- ==============================================

-- View for product catalog with inventory status
CREATE VIEW product_catalog AS
SELECT
    p.product_id,
    p.product_name,
    p.description,
    c.category_name,
    p.price,
    p.sku,
    pi.quantity_in_stock,
    CASE
        WHEN pi.quantity_in_stock = 0 THEN 'Out of Stock'
        WHEN pi.quantity_in_stock <= pi.low_stock_threshold THEN 'Low Stock'
        ELSE 'In Stock'
    END AS stock_status,
    p.is_active
FROM products p
JOIN categories c ON p.category_id = c.category_id
JOIN product_inventory pi ON p.product_id = pi.product_id;

-- View for customer order history
CREATE VIEW customer_order_history AS
SELECT
    o.order_id,
    o.customer_id,
    c.first_name,
    c.last_name,
    c.email,
    o.order_date,
    o.status,
    o.total_amount,
    COUNT(oi.order_item_id) AS items_count,
    SUM(oi.quantity) AS total_quantity
FROM orders o
JOIN customers c ON o.customer_id = c.customer_id
JOIN order_items oi ON o.order_id = oi.order_id
GROUP BY o.order_id, o.customer_id, c.first_name, c.last_name, c.email, o.order_date, o.status, o.total_amount;

-- ==============================================
-- CREATE STORED PROCEDURES
-- ==============================================

-- Procedure to place a new order
DELIMITER //

CREATE PROCEDURE PlaceOrder(
    IN p_customer_id INT,
    IN p_shipping_address_id INT,
    IN p_billing_address_id INT,
    IN p_items JSON,  -- JSON array of {product_id, quantity}
    IN p_shipping_method_id INT,
    IN p_coupon_code VARCHAR(50)
)
BEGIN
    DECLARE v_order_id INT;
    DECLARE v_subtotal DECIMAL(12, 2) DEFAULT 0;
    DECLARE v_discount DECIMAL(8, 2) DEFAULT 0;
    DECLARE v_total DECIMAL(12, 2);
    DECLARE v_shipping_cost DECIMAL(8, 2);
    DECLARE i INT DEFAULT 0;
    DECLARE v_product_id INT;
    DECLARE v_quantity INT;
    DECLARE v_unit_price DECIMAL(10, 2);
    DECLARE v_coupon_id INT;

    -- Start transaction
    START TRANSACTION;
    
    -- Calculate order subtotal
    WHILE i < JSON_LENGTH(p_items) DO
        SET v_product_id = JSON_EXTRACT(p_items, CONCAT('$[', i, '].product_id'));
        SET v_quantity = JSON_EXTRACT(p_items, CONCAT('$[', i, '].quantity'));
        
        -- Get current price
        SELECT price INTO v_unit_price FROM products WHERE product_id = v_product_id;
        
        -- Check inventory
        IF (SELECT quantity_in_stock FROM product_inventory WHERE product_id = v_product_id) < v_quantity THEN
            ROLLBACK;
            SIGNAL SQLSTATE '45000' SET MESSAGE_TEXT = 'Insufficient inventory for product';
        END IF;
        
        SET v_subtotal = v_subtotal + (v_unit_price * v_quantity);
        SET i = i + 1;
    END WHILE;
    
    -- Get shipping cost
    SELECT base_cost INTO v_shipping_cost FROM shipping_methods WHERE method_id = p_shipping_method_id;
    
    -- Apply coupon discount if provided
    IF p_coupon_code IS NOT NULL THEN
        SELECT coupon_id, discount_value INTO v_coupon_id, v_discount 
        FROM coupons 
        WHERE coupon_code = p_coupon_code 
          AND is_active = TRUE 
          AND CURRENT_DATE BETWEEN start_date AND end_date
          AND (usage_limit IS NULL OR times_used < usage_limit);
        
        IF v_coupon_id IS NOT NULL THEN
            -- Update coupon usage
            UPDATE coupons SET times_used = times_used + 1 WHERE coupon_id = v_coupon_id;
        END IF;
    END IF;
    
    -- Calculate total
    SET v_total = v_subtotal + v_shipping_cost - v_discount;
    
    -- Create order
    INSERT INTO orders (customer_id, shipping_address_id, billing_address_id, total_amount, shipping_cost)
    VALUES (p_customer_id, p_shipping_address_id, p_billing_address_id, v_total, v_shipping_cost);
    
    SET v_order_id = LAST_INSERT_ID();
    
    -- Add order items and update inventory
    SET i = 0;
    WHILE i < JSON_LENGTH(p_items) DO
        SET v_product_id = JSON_EXTRACT(p_items, CONCAT('$[', i, '].product_id'));
        SET v_quantity = JSON_EXTRACT(p_items, CONCAT('$[', i, '].quantity'));
        
        SELECT price INTO v_unit_price FROM products WHERE product_id = v_product_id;
        
        INSERT INTO order_items (order_id, product_id, quantity, unit_price)
        VALUES (v_order_id, v_product_id, v_quantity, v_unit_price);
        
        -- Update inventory
        UPDATE product_inventory 
        SET quantity_in_stock = quantity_in_stock - v_quantity 
        WHERE product_id = v_product_id;
        
        SET i = i + 1;
    END WHILE;
    
    -- Add coupon if applied
    IF v_coupon_id IS NOT NULL THEN
        INSERT INTO order_coupons (order_id, coupon_id, discount_applied)
        VALUES (v_order_id, v_coupon_id, v_discount);
    END IF;
    
    -- Add shipping info
    INSERT INTO order_shipping (order_id, shipping_method_id)
    VALUES (v_order_id, p_shipping_method_id);
    
    COMMIT;
    
    SELECT v_order_id AS new_order_id;
END //

DELIMITER ;

-- ==============================================
-- CREATE TRIGGERS
-- ==============================================

-- Trigger to update product inventory after order cancellation
DELIMITER //

CREATE TRIGGER after_order_cancelled
AFTER UPDATE ON orders
FOR EACH ROW
BEGIN
    IF OLD.status != 'cancelled' AND NEW.status = 'cancelled' THEN
        UPDATE product_inventory pi
        JOIN order_items oi ON pi.product_id = oi.product_id
        SET pi.quantity_in_stock = pi.quantity_in_stock + oi.quantity
        WHERE oi.order_id = NEW.order_id;
    END IF;
END //

DELIMITER ;

-- ==============================================
-- DATABASE METADATA QUERY
-- ==============================================

-- Show database structure summary
SELECT
    'Database Created Successfully' AS status,
    COUNT(*) AS total_tables,
    (SELECT COUNT(*) FROM information_schema.table_constraints
     WHERE table_schema = 'ecommerce_db' AND constraint_type = 'PRIMARY KEY') AS primary_keys,
    (SELECT COUNT(*) FROM information_schema.table_constraints
     WHERE table_schema = 'ecommerce_db' AND constraint_type = 'FOREIGN KEY') AS foreign_keys
FROM information_schema.tables
WHERE table_schema = 'ecommerce_db';

-- Show all tables with row counts
SELECT
    table_name,
    table_rows
FROM information_schema.tables
WHERE table_schema = 'ecommerce_db'
ORDER BY table_name;
