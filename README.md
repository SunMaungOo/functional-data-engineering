# What is Functional Data Engineering

Functional data engineering is the topic that I wanted to write about for a while. Functional data engineering is the concept based on software development's functional programming paradigm.The functional programming is mainly about creating ``pure function`` which does not have side effect (everything the function depend upon is given to function as parameter) because side effects make debugging difficult.

```python
class Math:
    def __init__(self,a):
        self.a = a
    
    def add(self,b):
        return self.a+b
```

In the above, the ``a`` value is being passed from the constructor (side effect) and we cannot say what the result of ``add(10)`` will be wwithout knowing what value of ``a`` was passed into the constructor.

```python
    def add(a,b):
        return a+b
```

But when we change it to pure function above ``add(10,6)`` we can know what the result will be because everything the function depend upon is given by parameter.

# Data Extraction Model

As a data engineer we usually deal with dimensions and facts. 

We usually extract the data from the source into the object storage staging layer and load the data from the object storage into the warehouse

``Source --> Staging (Minio,S3,Blob,--etc) ---> Warehouse``

## Facts

Let's say we want to ingest the customer transaction to our data warehouse.

The customer transaction will have 

```
transaction_id | account_id | amount | created_date | updated_date
```

We will create 2 pipelines (initial_load and delta_load) for it.

```python
def initial_load():
    customer_initial_transaction = load_customer_transaction_from_source()

    add_to_staging(customer_initial_transaction)

    customer_transaction = read_from_staging()

    create_customer_transaction_table_in_warehouse()

    load_to_warehouse(customer_transaction)

def delta_load():
    max_date = get_max_date_from_warehouse_customer_transaction()

    customer_delta_transaction = load_delta_customer_transaction_from_source(max_date)

    add_to_staging(customer_delta_transaction)

    customer_transaction = read_from_staging()

    load_to_warehouse(customer_transaction)

```

After running the initial and delta pipeline , the staging layer will usually have below file structure

```
    CustomerTransaction
        - initial 
            - customer-transaction.parquet
        - delta
            - yyyy
                - mm
                    - dd
                        - customer-transaction.parquet
```

There is nothing wrong with above file structure **until** we have 

1. Pipeline failure
2. Backfilling (due to late data arrival or insert due to business requirement)

In order to handle failure we have to write another recovery pipeline like below and write it with date we wanted to fix the customer transaction of

```python

def backfill_load(recovery_date):
    
    customer_fixed_transaction = load_customer_transaction_from_source_by_date(recovery_date)

    add_to_staging(customer_fixed_transaction)

    customer_transaction = read_from_staging()

    load_to_warehouse(customer_transaction)
```

with the object storage file structure will be

```
    CustomerTransaction
        - initial
            - customer-transaction.parquet
        - delta
            - yyyy
                - mm
                    - dd
                        - customer-transaction.parquet
        - backfill
            - yyyy
                - mm
                    - dd
                        - customer-transaction.parquet
```

### Applying Functional Concept (Facts)

In order to ingest customer transaction to warehouse we have to write 3 different pipeline

1. initial_load
2. delta_load
3. backfill

By applying the functional programming concept of pure function , we can reduce those 3 pipeline into the single idempotent pipeline. Because the pipeline is idempotent , we can run it in whatever order we want even in parallel.

```python
def functional_load(date):

    customer_transaction = load_customer_transaction_from_source_by_date(date)

    load_to_staging(customer_transaction)

    if not is_customer_transaction_table_exist_in_warehouse():
        create_customer_transaction_table_in_warehouse()

    customer_transaction = load_from_staging()

    insert_or_replace_date_if_exist(customer_transaction)

```

The above pipeline can be run in parallel because it is **Idempotent**.

It is idempotent because

1. It does not depend upon which date is already exist in warehouse table (like delta load)
2. Customer table is created if it does not already exist (we do not need to know whether the customer table is in the warehouse when we run the pipeline)
3. It does not care whether the data is already exist for that specific date

I want to go a little bit more about ``insert_or_replace_date_if_exist(customer_transaction)`` function. The function should delete the **whole** partition if it already exist and replace it with new data. The SQL should be something like

```sql

    DELETE FROM customer_transaction
    WHERE created_date = date;

    INSERT INTO customer_transaction
    (
        transaction_id,
        account_id,
        amount,
        created_date,
        updated_date
    )
    SELECT transaction_id,
    account_id,
    amount,
    created_date,
    updated_date
    FROM staging 
    WHERE created_date = date;
    
```

The staging file structure can be simplified into 

```
    CustomerTransaction
        - yyyy
            - mm
                - dd
                    - customer-transaction.parquet
```

We do not need to distinguish between ``initial-load`` ,``delta-load``, ``backfill`` because the ``functional_load`` pipeline work the same whether we are running ``initial-load`` , ``delta-load`` or ``backfilling`` the data.  

# Applying Functional Concept (Dimension)

For dimension data , we can apply the same concepts we apply for facts. There is no differences between dimension and facts in term of idempotent pipeline

# Historical data 

If you do not like to replace the file in the staging and wanted to preserve the old data for audit , historical , or SCD purpose we can structure the file in the staging layer as 

```
    CustomerTransaction
        - yyyy
            - mm
                - dd
                    - customer-transaction-timestamp.parquet
                    - customer-transaction-timestamp.parquet  
```

Every time we run the pipeline we will append timestamp in the file and the warehouse will just load data from the file which have the latest timestamp.

# Cons of functional approach

The disadvantage of the functional data engineering approach is the storage cost. Let say we wanted to backfill 10 new row from previous date. In normal data engineering approach we will just load that new 10 row but in the functional approach , we will load the 2010 row (``2000 (old row) + 10 (new row) ``). In functional approach it does not distinguish which rows are old and which rows are new and we need to load every rows.

# Summary

Functional data engineering is the approach which make the pipeline idempotent (which allows it to run in parallel and whatever order we want) while taking more storage space