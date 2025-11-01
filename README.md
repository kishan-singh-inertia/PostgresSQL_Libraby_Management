# PostgresSQL_Libraby_Management
Libraby Management Project in PostgresSQL.


# 📚 Library Management System

This project contains SQL scripts for creating and managing a **Library Management System** database.  
It includes table creation, CRUD operations, analytical queries, stored procedures, and triggers.

---

## 🗂️ Table Definitions

```sql
DROP TABLE IF EXISTS lib_branch;

Create table lib_branch(
			branch_id VARCHAR(10) PRIMARY KEY,
            manager_id VARCHAR(10),
            branch_address VARCHAR(30),
            contact_no VARCHAR(15)
        );


DROP TABLE IF EXISTS lib_emp;

Create table lib_emp(
			emp_id Varchar(10) PRIMARY KEY,
            emp_name VARCHAR(30),
            position VARCHAR(30),
            salary DECIMAL(10,2),
            branch_id VARCHAR(10),
            FOREIGN KEY(branch_id) REFERENCES lib_branch(branch_id)
        );


DROP TABLE IF EXISTS lib_members;
CREATE TABLE lib_members
(
            member_id VARCHAR(10) PRIMARY KEY,
            member_name VARCHAR(30),
            member_address VARCHAR(30),
            reg_date DATE
);



DROP TABLE IF EXISTS lib_books;
CREATE TABLE lib_books
(
            isbn VARCHAR(50) PRIMARY KEY,
            book_title VARCHAR(80),
            category VARCHAR(30),
            rental_price DECIMAL(10,2),
            status VARCHAR(10),
            author VARCHAR(30),
            publisher VARCHAR(30)
);




DROP TABLE IF EXISTS books_issued_status;
CREATE TABLE books_issued_status
(
            issued_id VARCHAR(10) PRIMARY KEY,
            issued_member_id VARCHAR(30),
            issued_book_name VARCHAR(80),
            issued_date DATE,
            issued_book_isbn VARCHAR(50),
            issued_emp_id VARCHAR(10),
            FOREIGN KEY (issued_member_id) REFERENCES lib_members(member_id),
            FOREIGN KEY (issued_emp_id) REFERENCES lib_emp(emp_id),
            FOREIGN KEY (issued_book_isbn) REFERENCES lib_books(isbn) 
);



DROP TABLE IF EXISTS books_return_status;
CREATE TABLE books_return_status
(
            return_id VARCHAR(10) PRIMARY KEY,
            issued_id VARCHAR(30),
            return_book_name VARCHAR(80),
            return_date DATE,
            return_book_isbn VARCHAR(50),
            FOREIGN KEY (return_book_isbn) REFERENCES lib_books(isbn)
);
```

---

## 🧾 CRUD Operations

### Q.1 Create a New Book Record
```sql
INSERT INTO lib_books VALUES('978-1-60129-456-2', 'To Kill a Mockingbird', 'Classic', 6.00, 'yes', 'Harper Lee', 'J.B. Lippincott & Co.');
```

### Update an Existing Member's Address
```sql
UPDATE lib_members 
SET member_address='Brooklyn 99' 
WHERE here member_id='C102';
```

### Delete a Record from the Issued Status Table
```sql
DELETE FROM books_issued_status 
WHERE issued_id='IS140';
```

### Retrieve All Books Issued by a Specific Employee
```sql
SELECT issued_book_name 
FROM books_issued_status
WHERE issued_member_id = 'C109';
```

### List Members Who Have Issued More Than One Book
```sql
SELECT issued_member_id
FROM books_issued_status 
GROUP BY 1
HAVING COUNT(*) > 1;
```

---

## 🧮 CTAS (Create Table As Select)

```sql
CREATE TABLE member_issued_cnt AS
SELECT member_id,member_name, COUNT(status.issued_id) AS issued_count
FROM books_issued_status as status
JOIN lib_members as mem
ON status.issued_member_id = mem.member_id
GROUP BY 1,2;
```

---

## 📖 Additional Queries

### Retrieve All Books in a Specific Category
```sql
SELECT book_title 
FROM lib_books
WHERE Category = 'Mystery';
```

### Find Total Rental Income by Category
```sql
SELECT books.category,SUM(books.rental_price)
FROM lib_books as books
JOIN books_issued_status as status
ON books.isbn=status.issued_book_isbn
GROUP BY 1;
```

### List Members Who Registered in the Last 900 Days
```sql
SELECT * FROM lib_members
WHERE reg_date >= CURRENT_DATE - INTERVAL '900 days';
```

### List Employees with Their Branch Manager's Name and Their Branch Details
```sql
SELECT emp_id,emp_name,emp.branch_id,manager_id,branch_address,contact_no 
from lib_emp as emp
JOIN lib_branch as bran
on emp.branch_id=bran.branch_id;
```

### Create a Table of Books with Rental Price Above a Certain Threshold
```sql
CREATE TABLE costly_books AS
select * from lib_books
where rental_price>7;
```

### Retrieve the List of Books Not Yet Returned
```sql
SELECT * 
from books_issued_status as st 
LEFT JOIN books_return_status as re 
on re.issued_id=st.issued_id
WHERE re.return_id is NULL;
```

---

## ⏰ Overdue Books Query

```sql
select me.member_id,me.member_name,st.issued_book_name,st.issued_date, CURRENT_DATE - st.issued_date as overdue_days from 
books_issued_status as st 
JOIN lib_members as me 
on st.issued_member_id=me.member_id
LEFT JOIN books_return_status as re 
on st.issued_id=re.issued_id
where re.return_date IS NULL 
AND (CURRENT_DATE - st.issued_date) < 100;
```

---

## 📚 Alter Table and Updates

```sql
ALTER TABLE books_return_status
ADD Column book_quality VARCHAR(15) DEFAULT('Good');

UPDATE books_return_status
SET book_quality = 'Damaged'
WHERE issued_id 
    IN ('IS112', 'IS117', 'IS118');
```

---

## 🧩 Stored Procedure – Update Book Status on Return

```sql
CREATE or REPLACE PROCEDURE insert_return(p_return_id VARCHAR(10),p_issued_id VARCHAR(10),p_book_quality VARCHAR(15))
LANGUAGE plpgsql
AS $$

DECLARE
v_isbn VARCHAR(50);
BEGIN

INSERT into books_return_status (return_id,issued_id,return_date,book_quality)
VALUES (p_return_id,p_issued_id,CURRENT_DATE,p_book_quality);

SELECT issued_book_isbn into v_isbn 
from books_issued_status
where issued_id=p_issued_id;

UPDATE lib_books
SET status = 'yes'
WHERE isbn = v_isbn;

END;
$$

Call insert_return('RS155','IS136','Good');
```

---

## 📊 Branch Performance Report

```sql
select bra.branch_id,bra.manager_id,
COUNT(st.issued_id) as cnt_books_issued, 
COUNT(re.return_id) as cnt_books_returned,
Sum(books.rental_price) as revenue
FROM books_issued_status as st 
JOIN lib_emp as emp
on st.issued_emp_id=emp.emp_id 
JOIN lib_branch as bra 
on bra.branch_id = emp.branch_id
LEFT JOIN books_return_status as re 
on re.issued_id = st.issued_id
JOIN lib_books as books
on books.isbn = st.issued_book_isbn
GROUP BY 1,2
ORDER BY 5 DESC;
```

---

## 🕒 Active Members (CTAS Example)

```sql
CREATE TABLE lib_active_members as
SELECT * from books_issued_status
WHERE (CURRENT_DATE - issued_date) < 365;
```

---

## 🏆 Top 3 Employees Who Processed Most Book Issues

```sql
select emp.emp_name,bra.branch_id,COUNT(st.issued_id)
from books_issued_status as st 
JOIN lib_emp as emp 
on emp.emp_id = st.issued_emp_id
JOIN lib_branch as bra
on bra.branch_id = emp.branch_id
GROUP BY 1,2
ORDER BY 3 DESC
LIMIT 3;
```

---

## ⚙️ Stored Procedure – Manage Book Status

```sql
CREATE or REPLACE PROCEDURE insert_issued(p_issued_id VARCHAR(10), p_issued_member_id VARCHAR(30), p_issued_book_isbn VARCHAR(30), p_issued_emp_id VARCHAR(10))
LANGUAGE plpgsql
AS $$

DECLARE
v_book_name VARCHAR(100);
v_status VARCHAR(10);

BEGIN

SELECT status into v_status 
FROM lib_books
WHERE isbn = p_issued_book_isbn;

IF v_status = 'yes' THEN
    SELECT status into v_book_name
    FROM lib_books
    WHERE isbn = p_issued_book_isbn;
    INSERT INTO books_issued_status 
    VALUES (p_issued_id, p_issued_member_id,v_book_name,
            CURRENT_DATE,p_issued_book_isbn, p_issued_emp_id);
    UPDATE lib_books
    SET STATUS = 'no'
    WHERE isbn = p_issued_book_isbn;
    RAISE NOTICE 'The book has been issued.';
ELSE
    RAISE NOTICE 'The book cannot be issued.';
END IF;

END;
$$

Call insert_issued('IS199','C119','978-0-375-50167-0','E109')
```

---

## 📈 Triggers and Procedures for Member Rating

```sql
ALTER TABLE books_return_status
ADD COLUMN return_rating INTEGER DEFAULT 10;

select COUNT(issued_id) as cnt_per_mem from books_issued_status where issued_member_id = 'C110';

CREATE OR REPLACE PROCEDURE upd_rating (p_return_rating INT,p_issued_id VARCHAR)
AS $$

DECLARE
v_cnt_per_mem INT DEFAULT (0);
v_current_rating DECIMAL;
v_updated_rating DECIMAL(10,2);
v_mem_id VARCHAR;

BEGIN

SELECT issued_member_id INTO v_mem_id 
FROM books_issued_status
WHERE issued_id = p_issued_id;

SELECT COUNT(issued_id) INTO v_cnt_per_mem 
FROM books_issued_status 
WHERE issued_member_id = v_mem_id;

SELECT mem_rating INTO v_current_rating 
FROM lib_members
WHERE member_id = v_mem_id;

v_updated_rating = ((v_current_rating * v_cnt_per_mem) + p_return_rating) / (v_cnt_per_mem + 1);

UPDATE lib_members
SET mem_rating = v_updated_rating
WHERE member_id = v_mem_id;

END;
$$ LANGUAGE plpgsql;

CREATE OR REPLACE FUNCTION fn_upd_rt() RETURNS TRIGGER AS $$
BEGIN
call upd_rating(NEW.return_rating,NEW.issued_id);
RETURN NEW;
END;
$$ LANGUAGE plpgsql;

drop procedure upd_rating;

CREATE OR REPLACE TRIGGER tr_upd_mem_rating
AFTER INSERT ON books_return_status
FOR EACH ROW
EXECUTE FUNCTION fn_upd_rt();
```

---

**End of Script**
