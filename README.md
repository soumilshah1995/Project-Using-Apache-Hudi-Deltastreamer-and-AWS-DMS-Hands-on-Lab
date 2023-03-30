
# Project : Using Apache Hudi Deltastreamer and AWS DMS Hands on Labs
![image](https://user-images.githubusercontent.com/39345855/228927370-f7264d4a-f026-4014-9df4-b063f000f377.png)



------------------------------------------------------------------
Video Tutorials 
Part 1: Project Overview : https://www.youtube.com/watch?v=D9L0NLSqC1s
Part 2: Aurora Setup : https://youtu.be/HR5A6iGb4LE

# Steps 
### Step 1:  Create Aurora Source Database and update the seetings to enable CDC on Postgres 
*  Read More : https://docs.aws.amazon.com/dms/latest/userguide/CHAP_Source.PostgreSQL.html
* Create new Create parameter group as shows in video and make sure to edit these two setting as shown in video part 2
```
rds.logical_replication  1
wal_sender_timeout   300000
```
once done please apply these to database and reboot your Database 


### Step 2: Run Python file to create a table called sales in public schema in aurora and lets populate some data into table 

Run python run.py
```

try:
    import os
    import logging

    from functools import wraps
    from abc import ABC, abstractmethod
    from enum import Enum
    from logging import StreamHandler

    import uuid
    from datetime import datetime, timezone
    from random import randint
    import datetime

    import sqlalchemy as db
    from faker import Faker
    import random
    import psycopg2
    import psycopg2.extras as extras
    from dotenv import load_dotenv

    load_dotenv(".env")
except Exception as e:
    raise Exception("Error: {} ".format(e))


class Logging:
    """
    This class is used for logging data to datadog an to the console.
    """

    def __init__(self, service_name, ddsource, logger_name="demoapp"):

        self.service_name = service_name
        self.ddsource = ddsource
        self.logger_name = logger_name

        format = "[%(asctime)s] %(name)s %(levelname)s %(message)s"
        self.logger = logging.getLogger(self.logger_name)
        formatter = logging.Formatter(format, )

        if logging.getLogger().hasHandlers():
            logging.getLogger().setLevel(logging.INFO)
        else:
            logging.basicConfig(level=logging.INFO)


global logger
logger = Logging(service_name="database-common-module", ddsource="database-common-module",
                 logger_name="database-common-module")


def error_handling_with_logging(argument=None):
    def real_decorator(function):
        @wraps(function)
        def wrapper(self, *args, **kwargs):
            function_name = function.__name__
            response = None
            try:
                if kwargs == {}:
                    response = function(self)
                else:
                    response = function(self, **kwargs)
            except Exception as e:
                response = {
                    "status": -1,
                    "error": {"message": str(e), "function_name": function.__name__},
                }
                logger.logger.info(response)
            return response

        return wrapper

    return real_decorator


class DatabaseInterface(ABC):
    @abstractmethod
    def get_data(self, query):
        """
        For given query fetch the data
        :param query: Str
        :return: Dict
        """

    def execute_non_query(self, query):
        """
        Inserts data into SQL Server
        :param query:  Str
        :return: Dict
        """

    def insert_many(self, query, data):
        """
        Insert Many items into database
        :param query: str
        :param data: tuple
        :return: Dict
        """

    def get_data_batch(self, batch_size=10, query=""):
        """
        Gets data into batches
        :param batch_size: INT
        :param query: STR
        :return: DICT
        """

    def get_table(self, table_name=""):
        """
        Gets the table from database
        :param table_name: STR
        :return: OBJECT
        """


class Settings(object):
    """settings class"""

    def __init__(
            self,
            port="",
            server="",
            username="",
            password="",
            timeout=100,
            database_name="",
            connection_string="",
            collection_name="",
            **kwargs,
    ):
        self.port = port
        self.server = server
        self.username = username
        self.password = password
        self.timeout = timeout
        self.database_name = database_name
        self.connection_string = connection_string
        self.collection_name = collection_name


class DatabaseAurora(DatabaseInterface):
    """Aurora database class"""

    def __init__(self, data_base_settings):
        self.data_base_settings = data_base_settings
        self.client = db.create_engine(
            "postgresql://{username}:{password}@{server}:{port}/{database}".format(
                username=self.data_base_settings.username,
                password=self.data_base_settings.password,
                server=self.data_base_settings.server,
                port=self.data_base_settings.port,
                database=self.data_base_settings.database_name
            )
        )
        self.metadata = db.MetaData()
        logger.logger.info("Auroradb connection established successfully.")

    @error_handling_with_logging()
    def get_data(self, query):
        self.query = query
        cursor = self.client.connect()
        response = cursor.execute(self.query)
        result = response.fetchall()
        columns = response.keys()._keys
        data = [dict(zip(columns, item)) for item in result]
        cursor.close()
        return {"statusCode": 200, "data": data}

    @error_handling_with_logging()
    def execute_non_query(self, query):
        self.query = query
        cursor = self.client.connect()
        cursor.execute(self.query)
        cursor.close()
        return {"statusCode": 200, "data": True}

    @error_handling_with_logging()
    def insert_many(self, query, data):
        self.query = query
        print(data)
        cursor = self.client.connect()
        cursor.execute(self.query, data)
        cursor.close()
        return {"statusCode": 200, "data": True}

    @error_handling_with_logging()
    def get_data_batch(self, batch_size=10, query=""):
        self.query = query
        cursor = self.client.connect()
        response = cursor.execute(self.query)
        columns = response.keys()._keys
        while True:
            result = response.fetchmany(batch_size)
            if not result:
                break
            else:
                items = [dict(zip(columns, data)) for data in result]
                yield items

    @error_handling_with_logging()
    def get_table(self, table_name=""):
        table = db.Table(table_name, self.metadata,
                         autoload=True,
                         autoload_with=self.client)

        return {"statusCode": 200, "table": table}


class DatabaseAuroraPycopg(DatabaseInterface):
    """Aurora database class"""

    def __init__(self, data_base_settings):
        self.data_base_settings = data_base_settings
        self.client = psycopg2.connect(
            host=self.data_base_settings.server,
            port=self.data_base_settings.port,
            database=self.data_base_settings.database_name,
            user=self.data_base_settings.username,
            password=self.data_base_settings.password,
        )

    @error_handling_with_logging()
    def get_data(self, query):
        self.query = query
        cursor = self.client.cursor()
        cursor.execute(self.query)
        result = cursor.fetchall()
        columns = [column[0] for column in cursor.description]
        data = [dict(zip(columns, item)) for item in result]
        cursor.close()
        _ = {"statusCode": 200, "data": data}

        return _

    @error_handling_with_logging()
    def execute(self, query, data):
        self.query = query
        cursor = self.client.cursor()
        cursor.execute(self.query, data)
        self.client.commit()
        cursor.close()
        return {"statusCode": 200, "data": True}

    @error_handling_with_logging()
    def get_data_batch(self, batch_size=10, query=""):
        self.query = query
        cursor = self.client.cursor()
        cursor.execute(self.query)
        columns = [column[0] for column in cursor.description]
        while True:
            result = cursor.fetchmany(batch_size)
            if not result:
                break
            else:
                items = [dict(zip(columns, data)) for data in result]
                yield items

    @error_handling_with_logging()
    def insert_many(self, query, data):
        self.query = query
        cursor = self.client.cursor()
        extras.execute_batch(cursor, self.query, data)
        self.client.commit()
        cursor.close()
        return {"statusCode": 200, "data": True}


class Connector(Enum):

    ON_AURORA_PYCOPG = DatabaseAurora(
        data_base_settings=Settings(
            port="5432",
            server="XXXXXXXXX",
            username="postgres",
            password="postgres",
            database_name="postgres",
        )
    )


def main():
    helper = Connector.ON_AURORA_PYCOPG.value
    import time

    states = ("AL", "AK", "AZ", "AR", "CA", "CO", "CT", "DE", "FL", "GA", "HI", "ID", "IL", "IN",
              "IA", "KS", "KY", "LA", "ME", "MD", "MA", "MI", "MN", "MS", "MO", "MT", "NE", "NV", "NH", "NJ",
              "NM", "NY", "NC", "ND", "OH", "OK", "OR", "PA", "RI", "SC", "SD", "TN", "TX", "UT", "VT", "VA",
              "WA", "WV", "WI", "WY")
    shipping_types = ("Free", "3-Day", "2-Day")

    product_categories = ("Garden", "Kitchen", "Office", "Household")
    referrals = ("Other", "Friend/Colleague", "Repeat Customer", "Online Ad")

    try:
        query = """
CREATE TABLE public.sales (
  invoiceid INTEGER,
  itemid INTEGER,
  category TEXT,
  price NUMERIC(10,2),
  quantity INTEGER,
  orderdate DATE,
  destinationstate TEXT,
  shippingtype TEXT,
  referral TEXT
);
        """
        helper.execute_non_query(query=query,)
        time.sleep(2)
    except Exception as e:
        print("Error",e)

    try:
        query = """
            ALTER TABLE execute_non_query.sales REPLICA IDENTITY  FULL
        """
        helper.execute(query=query)
        time.sleep(2)
    except Exception as e:
        pass

    for i in range(0, 100):

        item_id = random.randint(1, 100)
        state = states[random.randint(0, len(states) - 1)]
        shipping_type = shipping_types[random.randint(0, len(shipping_types) - 1)]
        product_category = product_categories[random.randint(0, len(product_categories) - 1)]
        quantity = random.randint(1, 4)
        referral = referrals[random.randint(0, len(referrals) - 1)]
        price = random.randint(1, 100)
        order_date = datetime.date(2016, random.randint(1, 12), random.randint(1, 28)).isoformat()
        invoiceid = random.randint(1, 20000)

        data_order = (invoiceid, item_id, product_category, price, quantity, order_date, state, shipping_type, referral)

        query = """INSERT INTO public.sales
                                            (
                                            invoiceid, itemid, category, price, quantity, orderdate, destinationstate,shippingtype, referral
                                            )
                                        VALUES (%s, %s, %s, %s, %s, %s, %s, %s, %s)"""

        helper.insert_many(query=query, data=data_order)


main()
```


### Step 3:  Create a DMS Replication Instance as shown in video 3
