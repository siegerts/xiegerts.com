+++
title = "Saving Scrapy Crawl Stats to PostgreSQL with a Custom Extension and SQLAlchemy"
description =  "Save the Scrapy spider crawl stats to a Postgres database using a custom extension and SQLAlchemy. We'll also update the existing database item pipeline to collect custom stats specific to the spider."

tags = [
  "scrapy",
  "jsonb",
  "python",
  "automation",  
  "web crawling",
  "web scraping",
  "sqlalchemy",
  "postgresql",
  "database",
  "corestats",
  "statscollector",
]

categories = [""]

series = "Web Scraping"

date = 2024-08-14T07:33:55-05:00


+++

The Scrapy crawl stat logs are useful for tracking and monitoring the performance of a spider. You've probably seen the stats log at the end of a spider crawl when they are dumped to the Scrapy log (or console) from memory.

An example of these `CoreStats` stats is below. Depending on the spider, settings, and other extensions, the stats may vary in your project.

```python
{
    "downloader/request_bytes": 1178,
    "downloader/request_count": 3,
    "downloader/request_method_count/GET": 3,
    "downloader/response_bytes": 132792,
    "downloader/response_count": 3,
    "downloader/response_status_count/200": 1,
    "downloader/response_status_count/301": 1,
    "downloader/response_status_count/404": 1,
    "elapsed_time_seconds": 2.278677,
    "finish_reason": "finished",
    "finish_time": datetime.datetime(
        2024, 8, 6, 1, 16, 25, 36923, tzinfo=datetime.timezone.utc
    ),
    "httpcompression/response_bytes": 941938,
    "httpcompression/response_count": 2,
    "log_count/INFO": 10,
    "log_count/WARNING": 1,
    "memusage/max": 95203328,
    "memusage/startup": 95203328,
    "response_received_count": 2,
    "robotstxt/request_count": 1,
    "robotstxt/response_count": 1,
    "robotstxt/response_status_count/404": 1,
    "scheduler/dequeued": 2,
    "scheduler/dequeued/memory": 2,
    "scheduler/enqueued": 2,
    "scheduler/enqueued/memory": 2,
    "start_time": datetime.datetime(
        2024, 8, 6, 1, 16, 22, 758246, tzinfo=datetime.timezone.utc
    ),
}
```

When the [STATS_DUMP](https://docs.scrapy.org/en/latest/topics/settings.html#stats-dump) Scrapy setting is enabled, which is the default, the stats are dumped into the log at the end of the spider run. These are the core stats (`CoreStats`). `CoreStats` uses the Scrapy [Stats Collector API](https://docs.scrapy.org/en/latest/topics/stats.html) to collect the stats during the spider run. The stats include the number of requests, responses, elapsed time, etc. Specifically, the default stats collector is the `MemoryStatsCollector`([`scrapy.statscollectors.MemoryStatsCollector`](https://docs.scrapy.org/en/latest/topics/settings.html#stats-class)) which collects the stats in memory. 

To get an idea of how the stats are collected, look at the [`scrapy.extensions.corestats` extension](https://docs.scrapy.org/en/latest/_modules/scrapy/extensions/corestats.html) in the Scrapy source code. The extension hooks into the Scrapy signals and collects the stats at specific points in the spider's lifecycle. These stats are also available in `crawler.stats` and can be accessed programmatically (more on this below). 

If you're like me, you want to save all of these to your database - it's a log of great info! Also, you can create additional workflows and dashboards to monitor the performance of your spiders :thinking:.

<div style="overflow-x: auto;">

| id | name      | created_dt                 | start_time                | finish_time                | finish_reason | elapsed_time_seconds | processed_items | saved_items | stats |
|----|-----------|----------------------------|---------------------------|----------------------------|---------------|----------------------|-----------------|-------------|-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
|  1 | `spider-name` | 2024-08-06 01:33:43.891065 | 2024-08-06 22:50:26.15658 | 2024-08-06 01:33:43.770914 | finished      |          9797.614334 |          100 |      953 | `{"memusage/max": 716877824, "log_count/INFO": 5, "log_count/ERROR": 4, "memusage/startup": 109961216, "log_count/WARNING": 1 ... "downloader/response_status_count/404": 1}` |

</div>

{{< note >}}

The values above are examples and may vary depending on the project. The `stats` column data is truncated for display purposes.

{{< /note >}}

In this post, we'll look at how to save the stats to a database, specifically Postgres using SQLAlchemy. We'll [create a Scrapy extension](https://docs.scrapy.org/en/latest/topics/extensions.html#writing-your-own-extension) to save the stats to the database when the spider is closed. Also, we'll update the existing database pipeline to collect custom stats specific to the spider (project).


## Saving the stats to a database


To save the stats to a database like Postgres, we can create a Scrapy extension. Extensions can hook into the Scrapy signals and run code at specific points in the spider lifecycle (like `CoreStats` above). To do this, we'll configure the extension to execute when the spider is closed using the `spider_closed` signal and associated callback method. 


A few things are needed first to persist the data in the database:

1. [SQLAlchemy](https://www.sqlalchemy.org/) model (`Job`) to store the crawl job stats
2. [Scrapy Extension](https://docs.scrapy.org/en/latest/topics/extensions.html) to save the stats to the database on spider close
3. [Scrapy Item Pipeline](https://docs.scrapy.org/en/latest/topics/item-pipeline.html) to collect custom stats - we'll update an existing database pipeline to collect the additional stats (`processed_items` and `saved_items`)
4. Update the project settings to include the extension and updated pipeline

The general project structure will look like this (this is an example and may vary depending on the project):

```
<project>/
├── <project>/
│   ├── spiders/
│   │   ├── __init__.py
│   │   └── <spider>.py
│   │
│   ├── __init__.py
│   ├── models.py
│   ├── extensions.py
│   ├── pipelines.py
│   └── settings.py
│
...
├── scrapy.cfg
└── requirements.txt
```

### Custom stats to collect

In addition to the core stats above, we'll collect two custome items specific to the spider (project) to the stats:

1. `processed_items` - items seen by the spider
2. `saved_items`- items saved to the database

These two values will be updated (incremented) in the item pipeline.


## Create a SQLAlchemy model to store the run stats

{{< note >}}

This example uses the `SQLAlchemy` ORM to create the model. The model is created in a separate file and imported into the extension. Postgres is used as the database in this example but can be replaced with another database like MySQL, SQLite, etc. You'll need to adjust the `stats` column to the appropriate column type for the database you are using if `JSONB` is not supported.

{{< /note >}}

Create the new model to store the crawl stats. An example model is shown below that logs the crawl stats to Postgres.

```python
# models.py

from sqlalchemy import Column, Integer, String, DateTime, Float
from sqlalchemy.dialects.postgresql import JSONB
from sqlalchemy.sql import func
from sqlalchemy.ext.declarative import declarative_base
# ... other imports

from <project> import settings

DeclarativeBase = declarative_base()


def db_connect():
    """
    Creates database connection using database settings from settings.py.
    Returns sqlalchemy engine instance
    """
    return create_engine(settings.DATABASE_URL)


class Job(DeclarativeBase):
    """
    Job model to store the crawl stats
    """

    __tablename__ = "jobs"

    id = Column(Integer, primary_key=True)
    name = Column(String)
    created_dt = Column(DateTime, default=func.now())
    start_time = Column(DateTime)
    finish_time = Column(DateTime)
    finish_reason = Column(String)
    elapsed_time_seconds = Column(Float)
    processed_items = Column(Integer)
    saved_items = Column(Integer)
    stats = Column(JSONB)


#... other models
```

The two custom metrics for the spider are:

```python
processed_items = Column(Integer)
saved_items = Column(Integer)
```

## Update the Item Pipeline to collect additional stats

Assuming that the project already has a database persistance pipeline, we'll update the pipeline to collect the additional stats. The pipeline will be updated to collect the `processed_items` and `saved_items` stats. 


```python
# pipelines.py
from sqlalchemy.orm import sessionmaker

from <project>.models import (
    CrawledItem,
    db_connect,
)

class DatabasePipeline:
    def __init__(self, stats):
        self.stats = stats #1

    @classmethod
    def from_crawler(cls, crawler):
        return cls(stats=crawler.stats) #2

    def open_spider(self, spider):
        """
        Initializes database connection and sessionmaker.
        Creates items table.
        """
        engine = db_connect()
        create_items_table(engine)
        self.Session = sessionmaker(bind=engine)

        self.stats.set_value("pipeline/database/processed_items", 0) #3
        self.stats.set_value("pipeline/database/saved_items", 0) #4

    def process_item(self, item, spider):
        """
        Process the item and store to database.
        """

        db = self.Session()

        instance = (
            db.query(CrawledItem).filter(CrawledItem.id == item["id"]).first()
        )

        # modify/update the item as needed
        if not instance:
            instance = CrawledItem(**item)
            db.add(instance)
        else:
            for key, value in item.items():
                setattr(instance, key, value)

        try:
            db.commit()
            self.stats.inc_value("pipeline/database/saved_items") #5
            return item
        except Exception as error:
            print(error)
            db.rollback()
            raise
        finally:
            self.stats.inc_value("pipeline/database/processed_items") #6
            db.close()


    def close_spider(self, spider):
        self.Session().close()


```
In the pipeline above, the following changes integrate the new stats collection. The `crawler.stats` object is (#1/2) passed to the pipeline in the `from_crawler` class method. This allows the pipeline to access the stats object and update the stats.

```python
# pipelines.py
# ...

def __init__(self, stats):
    self.stats = stats

@classmethod
def from_crawler(cls, crawler):
    return cls(stats=crawler.stats)

# ...
```

The `open_spider` method initializes the database connection and (#3/4) initializes the `processed_items` and `saved_items` stats to `0`.

Note, `pipeline/database/processed_items` and `pipeline/database/saved_items` are the keys used to store the stats in the `crawler.stats` dictionary. The naming convention structure (`pipeline/<pipeline-name>/<metric>`) is personal preference and not enforced. I tagged them this way to keep track of where the stats are coming from in the pipeline.

Then, the `process_item` method processes the item and stores it in the database. The `close_spider` method closes the session. (#5) `saved_items` is incremented when an item is saved to the database, and (#6)`processed_items` is incremented after an item has been processed.


## Create the extension to save the crawl job stats 

Extension time :100:. Now, we need to add the extension to bring this all together. The extension will save the stats to the database when the spider is closed.

In _extensions.py_, create a new Scrapy extension to save the crawl stats to the database. The extension will hook into the `spider_closed` signal and save the stats to the database. You'll need to import the `Job` model and the `db_connect` function from the models in your project.

The extension will look similar to this:

```python
# extensions.py
import logging
from sqlalchemy.orm import sessionmaker
from scrapy import signals
from scrapy.exceptions import NotConfigured

from <project>.models import (
    Job,
    db_connect,
)

logger = logging.getLogger(__name__)


class SaveCrawlStats:
    @classmethod
    def from_crawler(cls, crawler):
        # first check if the extension should be enabled and raise
        # NotConfigured otherwise
        if not crawler.settings.getbool("CRAWLERSAVESTATS_ENABLED"):
            raise NotConfigured

        ext = cls()
        crawler.signals.connect(ext.spider_closed, signal=signals.spider_closed)

        return ext

    def spider_closed(self, spider):
        engine = db_connect()
        self.Session = sessionmaker(bind=engine)
        db = self.Session()

        # get the crawl stats
        stats = spider.crawler.stats.get_stats()

        name = spider.name
        processed_items = stats.get("pipeline/database/processed_items", 0)
        saved_items = stats.get("pipeline/database/saved_items", 0)

        start_time = stats.get("start_time")
        finish_time = stats.get("finish_time")
        finish_reason = stats.get("finish_reason")
        elapsed_time_seconds = stats.get("elapsed_time_seconds")

        # remove from dict
        del stats["start_time"]
        del stats["finish_time"]
        del stats["finish_reason"]
        del stats["elapsed_time_seconds"]

        job = Job(
            name=name,
            start_time=start_time,
            finish_time=finish_time,
            finish_reason=finish_reason,
            elapsed_time_seconds=elapsed_time_seconds,
            processed_items=processed_items,
            saved_items=saved_items,
            stats=stats,
        )

        db.add(job)

        try:
            db.commit()

        except Exception as error:
            logger.error(error)
            db.rollback()
            raise

        finally:
            db.close()
            self.Session().close()

```

The name of the extension is `SaveCrawlStats`. First, the extension checks if it should be enabled by checking the `CRAWLERSAVESTATS_ENABLED` setting in the project settings. Then, the extension hooks into the `spider_closed` signal to execute the `spider_closed` callback when the spider finishes and closes. 

I created new columns for the "default" stats along with the two new metrics `processed_items` and `saved_items`. The `start_time`, `finish_time`, `finish_reason`, and `elapsed_time_seconds` are also saved to the database. Also, the spider name is passed in to capture the name of the spider that ran.

The `stats` column is a `JSONB` column that stores the stats (without `start_time`, `finish_time`, `finish_reason`, and `elapsed_time_seconds`) as a `JSONB` object. This allows for flexibility when storing the stats and querying the data later if other stats are added via extensions, middlewares, etc. Mainly, I want to be able to quickly spot-check that the spider crawl is running as expected. 

{{< note >}}

The setting name, `CRAWLERSAVESTATS_ENABLED`, should be updated to not conflict with other settings in the project. There is some guidance on naming settings in the [Scrapy extension documentation](https://docs.scrapy.org/en/latest/topics/extensions.html#extension-settings). 

{{< /note >}}

When the method is called, the extension connects to the database, gets the stats from the spider, and saves the stats to the database. The stats are saved to the `Job` model created earlier. The `processed_items` and `saved_items` stats are retrieved from the `crawler.stats` object and saved to the database.

{{< warning >}}

Make sure that the `Job` model exists in the database!

{{< /warning >}}

## Update the project settings to include the new extension

Now that the extension and pipeline are created, update the project settings (_settings.py_) to include both options. The extension should be included in the `EXTENSIONS` setting, and the pipeline should be included in the `ITEM_PIPELINES` setting.

Also, make sure that the `CRAWLERSAVESTATS_ENABLED` is enabled by setting the value to `True`.

```python
# settings.py

CRAWLERSAVESTATS_ENABLED = True

EXTENSIONS = {
    ...
    '<project>.extensions.SaveCrawlStats': 500,
}

# ...

ITEM_PIPELINES = {
    ...
    '<project>.pipelines.DatabasePipeline': 500,
}

```

{{< note >}}

The `500` value is the priority of the extension and pipeline. The priority determines the order in which the extension and pipeline are executed. The lower the number, the higher the priority. The extension and pipeline are executed in order of priority. Adjust as needed.

{{< /note >}}

Now, after the next spider run, the stats will be saved to the database! You will start to build up a log of the spider runs and can create additional workflows and dashboards to monitor the performance of the spiders. You can also configure these settings in the spider settings using `custom_settings` if you want to enable/disable the extension and pipeline for specific spiders.


## Stats Collector alternative

An alternative approach to the custom extension is to create another stats collector that inherits from the `StatsCollector` class and overrides the `persist_stats` method. This method is called when the spider is closed and the stats are dumped to the log. You can override this method to save the stats to the database. In this approach, you can reuse most of the new extension code and update the `persist_stats` method.

In the stats collector module, you can see how the [`MemoryStatsCollector`](https://docs.scrapy.org/en/latest/_modules/scrapy/statscollectors.html#MemoryStatsCollector) is implemented using the `_persist_stats` method. You can use this as a reference to implement your own stats collector.

If you take this approach then you'll also need to update the `STATS_CLASS` setting in the project settings to use the new stats collector. Personally, I don't think there is a right or wrong way to do this - it's more about what works best for your project.

---

Hopefully this helps :rocket: If you have any questions, reach out!



