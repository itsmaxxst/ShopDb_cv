# ShopDb_cv

Qua ci sono tutte le query fatte per creare questo db

CREATE DATABASE shop_cv;

-----Tables-----

CREATE TABLE  Customers (
customer_id INTEGER PRIMARY KEY AUTO_INCREMENT,
name VARCHAR(100) NOT NULL,
email VARCHAR(255) UNIQUE CHECK(email != ''),
phone VARCHAR(20) UNIQUE CHECK(LENGTH(phone) >= 10)
);

CREATE TABLE  Orders (
order_id INTEGER PRIMARY KEY AUTO_INCREMENT,
customer_id INTEGER NOT NULL,
order_date DATETIME NOT NULL,
total_amount DECIMAL(10,2) NOT NULL DEFAULT 0.00,
FOREIGN KEY(customer_id) REFERENCES Customers(customer_id) ON DELETE CASCADE
);

CREATE TABLE  Products (
product_id INTEGER PRIMARY KEY AUTO_INCREMENT,
name VARCHAR(255) NOT NULL,
description VARCHAR(255) NOT NULL,
price DECIMAL(10,2) NOT NULL DEFAULT 0.00
);

CREATE TABLE  OrderItems (
order_item_id INTEGER PRIMARY KEY AUTO_INCREMENT,
order_id INTEGER NOT NULL,
product_id INTEGER NOT NULL,
quantity INTEGER,
unit_price DECIMAL(10,2) NOT NULL DEFAULT 0.00,
FOREIGN KEY(order_id) REFERENCES Orders(order_id) ON DELETE CASCADE,
FOREIGN KEY(product_id) REFERENCES Products(product_id) ON DELETE CASCADE
);

-----Triggers-----

DELIMITER $$

CREATE TRIGGER after_order_item_insert
AFTER INSERT ON OrderItems
FOR EACH ROW
BEGIN
    DECLARE new_total DECIMAL(10,2);

    SELECT SUM(quantity * unit_price)
    INTO new_total
    FROM OrderItems
    WHERE order_id = NEW.order_id;

    UPDATE Orders
    SET total_amount = new_total
    WHERE order_id = NEW.order_id;
END $$

DELIMITER ;

DELIMITER $$

CREATE TRIGGER after_order_item_update
AFTER UPDATE ON orderItems
FOR EACH ROW
BEGIN
    DECLARE new_total DECIMAL(10,2);

    SELECT SUM(quantity * unit_price)
    INTO new_total
    FROM orderItems
    WHERE order_id = OLD.order_id;

    UPDATE orders
    SET total_amount = new_total
    WHERE order_id = OLD.order_id;
END $$

DELIMITER ;

DELIMITER $$

CREATE TRIGGER after_order_item_delete
AFTER DELETE ON orderItems
FOR EACH ROW
BEGIN
    DECLARE new_total DECIMAL(10,2);

    SELECT SUM(quantity * unit_price)
    INTO new_total
    FROM orderItems
    WHERE order_id = OLD.order_id;

    UPDATE orders
    SET total_amount = new_total
    WHERE order_id = OLD.order_id;
END $$

DELIMITER ;

DELIMITER $$

CREATE TRIGGER before_order_item_insert
BEFORE INSERT ON orderitems
FOR EACH ROW
BEGIN
	DECLARE available_quantity INTEGER;
    
    SELECT quantity
    INTO available_quantity
    FROM products
    WHERE order_id = NEW.order_id;
    
    IF available_quantity < NEW.quantity THEN 
    SIGNAL SQLSTATE '45000' SET MESSAGE_TEXT = 'Not enough stock available for the product';
    END IF;
    
END $$

DELIMITER ;

-----Queries-----

SELECT 
	c.name AS customer,
	p.name AS product,
    oi.quantity,
    oi.unit_price AS price,
    o.order_date AS date,
    o.total_amount AS total
FROM orders o
INNER JOIN customers c
	ON o.customer_id = c.customer_id
INNER JOIN orderitems oi 
	ON o.order_id = oi.order_id
INNER JOIN products p
	ON oi.product_id = p.product_id
WHERE c.customer_id = 1;

SELECT 
	p.name AS product,
	SUM(oi.quantity) AS monthly_quantity,
	SUM(oi.quantity * p.price) AS monthly_total
FROM orderitems oi
INNER JOIN orders o 
	ON oi.order_id = o.order_id
INNER JOIN products p 
	ON p.product_id = oi.product_id
WHERE o.order_date >= '2024-02-01' 
AND o.order_date < '2024-03-01'
GROUP BY p.name
ORDER BY monthly_quantity DESC;

SELECT 
	SUM(oi.quantity) AS monthly_quantity,
	SUM(oi.quantity * p.price) AS monthly_total
FROM orderitems oi
INNER JOIN orders o 
	ON oi.order_id = o.order_id
INNER JOIN products p 
	ON p.product_id = oi.product_id
WHERE o.order_date >= '2024-02-01' 
AND o.order_date < '2024-03-01';

-----Stored procedures-----

DELIMITER $$

CREATE PROCEDURE CreateOrder(
    IN p_customer_id INT, 
    IN p_order_date DATETIME,
    IN p_items JSON
)
BEGIN
    DECLARE v_order_id INT;
    DECLARE v_product_id INT;
    DECLARE v_quantity INT;
    DECLARE v_stock INT;
    DECLARE v_unit_price DECIMAL(10,2);
    DECLARE done INT DEFAULT 0;

    DECLARE cur CURSOR FOR 
        SELECT 
            JSON_UNQUOTE(JSON_EXTRACT(t.value, '$.product_id')), 
            JSON_UNQUOTE(JSON_EXTRACT(t.value, '$.quantity'))
        FROM JSON_TABLE(p_items, '$[*]' COLUMNS (
            value JSON PATH '$'
        )) t;

    DECLARE CONTINUE HANDLER FOR NOT FOUND SET done = 1;

    START TRANSACTION;

    INSERT INTO orders (customer_id, order_date, total_amount) 
    VALUES (p_customer_id, p_order_date, 0);

    SET v_order_id = LAST_INSERT_ID(); 

    OPEN cur;

    read_loop: LOOP
        FETCH cur INTO v_product_id, v_quantity;
        IF done THEN 
            LEAVE read_loop;
        END IF;

        SELECT stock, price INTO v_stock, v_unit_price FROM products WHERE product_id = v_product_id;

        IF v_stock < v_quantity THEN
            ROLLBACK; 
            SIGNAL SQLSTATE '45000' 
            SET MESSAGE_TEXT = 'Not enough stock for product';
        END IF;

        INSERT INTO orderitems (order_id, product_id, quantity, unit_price) 
        VALUES (v_order_id, v_product_id, v_quantity, v_unit_price);

        UPDATE products 
        SET stock = stock - v_quantity 
        WHERE product_id = v_product_id;

    END LOOP;
    
    CLOSE cur;

    UPDATE orders 
    SET total_amount = (SELECT SUM(quantity * unit_price) FROM orderitems WHERE order_id = v_order_id)
    WHERE order_id = v_order_id;

    COMMIT;

END $$

DELIMITER ;

DELIMITER $$

CREATE PROCEDURE ShipOrder(IN p_order_id INT)
BEGIN
    
    DECLARE v_order_exists INT;
    DECLARE v_product_id INT;
    DECLARE v_quantity INT;
    DECLARE v_stock INT;
    DECLARE done INT DEFAULT 0;

    DECLARE cur CURSOR FOR 
        SELECT product_id, quantity 
        FROM orderitems 
        WHERE order_id = p_order_id;

    DECLARE CONTINUE HANDLER FOR NOT FOUND SET done = 1;

    START TRANSACTION;

    SELECT COUNT(*) INTO v_order_exists FROM orders WHERE order_id = p_order_id;
    
    IF v_order_exists = 0 THEN
        ROLLBACK;
        SIGNAL SQLSTATE '45000' 
        SET MESSAGE_TEXT = 'Order does not exist';
    END IF;

    OPEN cur;

    read_loop: LOOP
        FETCH cur INTO v_product_id, v_quantity;
        IF done THEN 
            LEAVE read_loop;
        END IF;

        SELECT stock INTO v_stock FROM products WHERE product_id = v_product_id;

        IF v_stock < v_quantity THEN
            ROLLBACK;
            SIGNAL SQLSTATE '45000' 
            SET MESSAGE_TEXT = 'Not enough stock for product';
        END IF;
    END LOOP;

    CLOSE cur;

    SET done = 0; 
    OPEN cur;

    update_loop: LOOP
        FETCH cur INTO v_product_id, v_quantity;
        IF done THEN 
            LEAVE update_loop;
        END IF;

        UPDATE products 
        SET stock = stock - v_quantity 
        WHERE product_id = v_product_id;
    END LOOP;

    CLOSE cur;

    UPDATE orders 
    SET status = 'shipped' 
    WHERE order_id = p_order_id;

    COMMIT;
END $$

DELIMITER ;









