1.	CREATE INSTANCE WITH EXISTING VPC AND SG WITH RUNNING CONTAINER OF MYSQL AND  ENTER DATA IN DATABASE USING TERRAFORM IN VSCODE

STEP1 :login with IAM user in vscode using access key and secret key 

STEP2: create directory and change in directory

STEP 3:create files like main.tf, variables.tf,security.tf,output.tf,terraform.vartf,sri.py,

STEP 4:give command for execution
                Terraform init
              Terraform validate
              Terraform plan
              Terraform apply

STEP 5:check the instance created or not and login that instance using ssh 
             Connection ssh -i "C:\Users\DELL\.ssh\keypair.pem" ec2-user@ec2-13-202-80-91.ap-south-1.compute.amazonaws.com

STEP 6:check the docker container and databases are present in inside the 
             Container 
             Docker ps  ---------- if container present 
             docker exec -it containerid bash  -----go inside the container
            after going inside the container login mysql
                                    user : mysql -u root -p
                                    password: root
                                 to check----show databases;  if the databeses are not presented then create database with python in terraform 

STEP 7: install python on windows and vscode and install pip  in vs code
         Command ---   python -m pip install mysql-connector-python

STEP 8: create python.py file using python code and excute the python file
              Using command python3 pythonfile name 

STEP 9: check the database created or not in  if it is created then it will show 
             The data
             Then we give the commands
             Show databases; ----then it will show your database name then select your database by using command
             Use databasename; ---then it will change change databse
            Select * from tablename;then it will show the table with given data
        
     MAIN.tf
     
provider "aws" {
  region = var.region
}

# Fetch default VPC
data "aws_vpc" "default" {
  default = true
}

# Fetch all subnets in default VPC
data "aws_subnets" "default" {
  filter {
    name   = "vpc-id"
    values = [data.aws_vpc.default.id]
  }
}

# EC2 Instance
resource "aws_instance" "mysql_instance" {
  ami                    = var.ami
  instance_type          = var.instance_type
  subnet_id              = data.aws_subnets.default.ids[0]
  key_name               = var.key_name
  vpc_security_group_ids = [aws_security_group.mysql_sg.id]
  associate_public_ip_address = true

  user_data = <<-EOF
              #!/bin/bash
              yum update -y
              yum install docker python3 -y
              systemctl start docker
              systemctl enable docker
              usermod -aG docker ec2-user
              docker run -d --name mysql -e MYSQL_ROOT_PASSWORD=root -p 3306:3306 mysql:latest
              EOF

  tags = {
    Name = "Terraform-MySQL"
  }
}

     Variables.tf

variable "region" {
  default = "ap-south-1"
}

variable "instance_type" {
  default = "t3.micro"
}

variable "ami" {
  # Amazon Linux 2 AMI - ap-south-1
  default = "ami-071edcf66e251a763"
}

variable "key_name" {
  description = "Your AWS key pair name"
  type        = string
}

    Terraform.tfvars

region        = "ap-south-1"
instance_type = "t3.micro"
key_name      = "eeeeeee"
 sri.py
import mysql.connector
from mysql.connector import Error
import time

DB_HOST = "13.202.80.91"  # Public IP of your EC2
DB_USER = "root"
DB_PASSWORD = "root"
DB_NAME = "company"

# ----------------------------------------
# Connect with Retry Logic
# ----------------------------------------
def get_connection(retries=10, delay=5):
    for attempt in range(1, retries + 1):
        try:
            print(f"[INFO] Connecting to MySQL (Attempt {attempt}/{retries})...")
            conn = mysql.connector.connect(
                host=DB_HOST,
                user=DB_USER,
                password=DB_PASSWORD
            )
            print("[SUCCESS] Connected to MySQL server.")
            return conn
        except Error as e:
            print(f"[WARN] Connection failed: {e}")
            time.sleep(delay)
    raise Exception("[ERROR] Could not connect to MySQL after multiple attempts.")

# ----------------------------------------
# Ensure DB & Table Exist
# ----------------------------------------
def setup_database(cursor):
    cursor.execute(f"CREATE DATABASE IF NOT EXISTS {DB_NAME};")
    cursor.execute(f"USE {DB_NAME};")
    
    cursor.execute("""
        CREATE TABLE IF NOT EXISTS employees (
            id INT AUTO_INCREMENT PRIMARY KEY,
            name VARCHAR(50) UNIQUE,
            department VARCHAR(50)
        );
    """)
    print("[INFO] Database & table ready.")

# ----------------------------------------
# Insert Employees if not exist
# ----------------------------------------
def insert_employees(cursor, conn):
    employees = [
        ("srikanth", "Engineering"),
        ("vijay", "Finance"),
        ("siva", "HR"),
        ("Abhi", "Security"),
        ("raj", "Education"),
    ]

    sql = "INSERT IGNORE INTO employees (name, department) VALUES (%s, %s);"

    cursor.executemany(sql, employees)
    conn.commit()
    print("[INFO] Data inserted (duplicates skipped).")

# ----------------------------------------
# Display Data
# ----------------------------------------
def display_employees(cursor):
    cursor.execute("SELECT * FROM employees;")
    rows = cursor.fetchall()

    print("\n--- Employee Table ---")
    for row in rows:
        print(f"ID={row[0]}, Name={row[1]}, Dept={row[2]}")
    print("----------------------\n")

# ----------------------------------------
# Main Execution
# ----------------------------------------
try:
    conn = get_connection()
    cursor = conn.cursor()

    setup_database(cursor)
    insert_employees(cursor, conn)
    display_employees(cursor)

except Exception as e:
    print(f"[FATAL] Script failed: {e}")

finally:
    if cursor:
        cursor.close()
    if conn:
        conn.close()
    print("[INFO] Script finished.")

output.tf
output "instance_public_ip" {
  value = aws_instance.mysql_instance.public_ip
}

output "vpc_id" {
  value = data.aws_vpc.default.id
}

output "subnet_id" {
  value = data.aws_subnets.default.ids[0]
}

output "security_group_id" {
  value = aws_security_group.mysql_sg.id
}

