以下是在 Ubuntu 中安裝 Apache Hive 的完整步驟，包括配置和測試。請按照以下指令逐步執行。

---

### **步驟 1：安裝必備環境**

登入至 hadoop 帳號
```bash
sudo su - hadoop
```
#### **1.1 更新系統**
```bash
sudo apt update && sudo apt upgrade -y
```

#### **1.2 安裝 Java**
檢查 Java 版本：
```bash
java -version
```
應顯示類似於 `openjdk version "8.x.x"` 的輸出。


#### **1.3 測試 yarn 啟動 mapreduce 執行任務**
```bash
yarn jar $HADOOP_HOME/share/hadoop/mapreduce/hadoop-mapreduce-examples-*.jar pi 2 10
```

### **步驟 2：下載並安裝 Hive**
#### **2.1 下載 Hive**
從官方 Apache 網站下載 Hive：
```bash
wget https://archive.apache.org/dist/hive/hive-3.1.3/apache-hive-3.1.3-bin.tar.gz
```

#### **2.2 解壓並移動到目標目錄**
```bash
tar -xvzf apache-hive-3.1.3-bin.tar.gz
sudo mv apache-hive-3.1.3-bin /usr/local/hive
```

#### **2.3 配置 Hive 環境變數**
編輯 `~/.bashrc`，添加以下內容：
```bash
vim ~/.bashrc
```
```bash
export HIVE_HOME=/usr/local/hive
export PATH=$PATH:$HIVE_HOME/bin
```
應用環境變數：
```bash
source ~/.bashrc
```

---

### **步驟 3：配置 Hive**
#### **3.1 配置 Metastore**
Hive 使用 Metastore 儲存表結構信息，需配置 MySQL 作為 Metastore。

1. 安裝 MySQL：
   ```bash
   sudo apt install mysql-server -y
   ```

2. 啟動 MySQL 並執行安全設置：
   ```bash
   sudo service mysql start
   sudo mysql_secure_installation
   ```

3. 創建 Hive 的 Metastore 數據庫：
   進入 MySQL：
   ```bash
   sudo mysql -u root -p
   ```
   執行以下命令創建數據庫和 Hive 用戶：
   ```sql
   CREATE DATABASE metastore;
   CREATE USER 'hiveuser'@'localhost' IDENTIFIED BY 'hivepassword';
   GRANT ALL PRIVILEGES ON metastore.* TO 'hiveuser'@'localhost';
   FLUSH PRIVILEGES;
   ```
   配置 MySQL root 用戶：
   ```sql
   CREATE USER 'root'@'localhost' IDENTIFIED BY 'hadoop_csim';
   GRANT ALL PRIVILEGES ON *.* TO 'root'@'localhost' WITH GRANT OPTION;
   ALTER USER 'root'@'localhost' IDENTIFIED WITH mysql_native_password BY 'hadoop_csim';
   FLUSH PRIVILEGES;
   ```
   離開資料庫
   ```sql
   exit;
   ```
#### **3.2 配置 Hive 的 Metastore**
編輯 Hive 配置文件 `hive-site.xml`：
```bash
sudo vim /usr/local/hive/conf/hive-site.xml
```

添加以下內容：
```xml
<configuration>
  <property>
    <name>javax.jdo.option.ConnectionURL</name>
    <value>jdbc:mysql://localhost/metastore</value>
    <description>JDBC connect string for a JDBC metastore</description>
  </property>

  <property>
    <name>javax.jdo.option.ConnectionDriverName</name>
    <value>com.mysql.cj.jdbc.Driver</value>
    <description>Driver class name for a JDBC metastore</description>
  </property>

  <property>
    <name>javax.jdo.option.ConnectionUserName</name>
    <value>hiveuser</value>
  </property>

  <property>
    <name>javax.jdo.option.ConnectionPassword</name>
    <value>hivepassword</value>
  </property>
</configuration>
```

#### **3.3 添加 MySQL 驅動**
下載 MySQL 驅動並將其放入 Hive 的 `lib` 目錄：
```bash
wget https://repo1.maven.org/maven2/com/mysql/mysql-connector-j/8.0.33/mysql-connector-j-8.0.33.jar
sudo cp mysql-connector-j-8.0.33.jar /usr/local/hive/lib/
```
下載 Hive JDBC 驅動並將其放入 Hive 的 `lib` 目錄和上載至 `HDFS`：
```bash
wget https://repo1.maven.org/maven2/org/apache/hive/hive-jdbc-handler/3.1.2/hive-jdbc-handler-3.1.2.jar
sudo cp hive-jdbc-handler-3.1.2.jar /usr/local/hive/lib/

hadoop fs -put hive-jdbc-handler-3.1.2.jar /
```
---

### **步驟 4：初始化 Metastore**
執行以下命令初始化 Metastore：
```bash
schematool -initSchema -dbType mysql
```

---

### **步驟 5：啟動 Hive 並測試**
#### **5.1 啟動 Hive**
進入 Hive 命令行界面：
```bash
hive
```

#### **5.2 創建測試表**
執行以下 HiveQL 語句創建測試表：
```sql
CREATE TABLE test_table (
    id INT,
    name STRING
)
STORED AS TEXTFILE;

INSERT INTO test_table VALUES (1, 'Alice'), (2, 'Bob');

SELECT * FROM test_table;
```

如果成功返回查詢結果，則表明 Hive 安裝與配置完成。

---

### **常見問題**
1. **`ClassNotFoundException: com.mysql.cj.jdbc.Driver`**
   - 確保 MySQL 驅動已正確放置於 `/usr/local/hive/lib/` 目錄。

2. **`Permission Denied` 錯誤**
   - 確保 Hadoop 的目錄權限正確：
     ```bash
     sudo chmod -R 777 /tmp/hive
     sudo chmod -R 777 /user/hive/warehouse
     ```

3. **Metastore 啟動失敗**
   - 確保 MySQL 啟動且用戶名密碼配置正確。

---