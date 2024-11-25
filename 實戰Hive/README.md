
#### **3.2 配置 Hive 的 Metastore**   
1. 下載北風數據庫的 SQL 文件：
   ```bash
   wget https://storage.googleapis.com/google-code-archive-downloads/v2/code.google.com/northwindextended/Northwind.MySQL5.sql
   ```

2. 登錄 MySQL 並創建數據庫：
   ```bash
   sudo mysql -u root -p
   ```

3. 在 MySQL 中執行以下命令：
   ```sql
   CREATE DATABASE northwind;
   USE northwind;
   SOURCE Northwind.MySQL5.sql;
   SHOW TABLES;


以下是實現查詢每個客戶總訂單數量的步驟和相關指令：

---

### **步驟 1: 使用 Sqoop 匯出 `customers` 表至 HDFS**

1. **匯出 `customers` 表到 HDFS**:
   ```bash
   sqoop import \
   --connect jdbc:mysql://localhost/northwind \
   --username root --password "hadoop_csim" \
   --table customers \
   --target-dir /user/hadoop/northwind/customers \
   --m 1 \
   --driver com.mysql.cj.jdbc.Driver
   ```

2. **檢查 HDFS 中的數據**:
   ```bash
   hadoop fs -ls /user/hadoop/northwind/customers
   hadoop fs -cat /user/hadoop/northwind/customers/part-m-00000
   ```

---

### **步驟 2: 使用 Hive 讀取 HDFS 的 `customers` 數據並創建表格**

1. **創建 `customers` 表**:
   ```sql
   CREATE TABLE customers (
       CustomerID STRING,
       CompanyName STRING,
       ContactName STRING,
       ContactTitle STRING,
       Address STRING,
       City STRING,
       Region STRING,
       PostalCode STRING,
       Country STRING,
       Phone STRING,
       Fax STRING
   )
   ROW FORMAT DELIMITED
   FIELDS TERMINATED BY ','
   STORED AS TEXTFILE;
   ```

2. **將 HDFS 數據加載到 `customers` 表**:
   ```sql
   LOAD DATA INPATH '/user/hadoop/northwind/customers' INTO TABLE customers;
   ```

3. **檢查 `customers` 表中的數據**:
   ```sql
   SELECT * FROM customers LIMIT 10;
   ```

---

### **題目 3: 查詢銷售數據**
#### **需求**
從 `Orders` 表中查詢每個客戶的總訂單數量。
### **步驟 1: 使用 Hive 讀取 MySQL 中的 `orders` 表數據並創建表格**

1. **將 `orders` 表作為外部表連接到 MySQL**:
   ```sql
   add jar hdfs://hive-jdbc-handler-3.1.2.jar;
   
   CREATE EXTERNAL TABLE orders (
       OrderID INT,
       CustomerID STRING,
       EmployeeID INT,
       OrderDate STRING,
       RequiredDate STRING,
       ShippedDate STRING,
       ShipVia INT,
       Freight DOUBLE,
       ShipName STRING,
       ShipAddress STRING,
       ShipCity STRING,
       ShipRegion STRING,
       ShipPostalCode STRING,
       ShipCountry STRING
   )
   STORED BY "org.apache.hadoop.hive.jdbc.HiveStorageHandler"
   TBLPROPERTIES (
       "hive.sql.database.type" = "MYSQL",
       "hive.jdbc.url"="jdbc:mysql://localhost:3306/northwind",
       "hive.jdbc.username"="root",
       "hive.jdbc.password"="hadoop_csim",
       "hive.jdbc.table"="Orders"
   );
   ```

2. **檢查 `orders` 表數據**:
   ```sql
   SELECT * FROM orders LIMIT 10;
   ```

---

### **步驟 2: 查詢每個客戶的總訂單數量**

1. **使用 HiveQL 查詢數據**:
   ```sql
   SELECT
       c.CustomerID,
       c.CompanyName,
       COUNT(o.OrderID) AS TotalOrders
   FROM
       customers c
   JOIN
       orders o
   ON
       c.CustomerID = o.CustomerID
   GROUP BY
       c.CustomerID, c.CompanyName
   ORDER BY
       TotalOrders DESC;
   ```

2. **執行結果**:
   - 顯示每個客戶的公司名稱和總訂單數量。

---

### **附加說明**
- 如果遇到 MySQL 連接問題，檢查 MySQL 的授權：
   ```sql
   GRANT ALL PRIVILEGES ON northwind.* TO 'root'@'localhost' IDENTIFIED BY 'your_mysql_password';
   FLUSH PRIVILEGES;
   ```

- 使用 HiveQL 的 SQL 風格語法，可以直接從分散式存儲（HDFS）和關係型數據庫（MySQL）中進行結合分析，方便進行大規模數據處理。