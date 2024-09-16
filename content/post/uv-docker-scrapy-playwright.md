+++
title = "Dockerfile with uv for Scrapy and Playwright"
description = "Dockerfile with uv for Scrapy and Playwright for web scraping and automation"

tags = [
    "python",
    "uv",
    "docker",
    "virtual environment",
    "web scraping",
    "scrapy",
    "playwright",
    "javascript",
    "automation",
    "headless browser",
    "web crawler",
    "chromium",
    "firefox"
]

categories = ["Development"]
date = 2024-09-15T15:25:16-04:00

series = "Web Scraping"

+++

These are some Dockerfile examples for setting up a Scrapy project with [uv](https://docs.astral.sh/uv/) and [Playwright](/post/infinite-scroll-scrapy-playwright/) for quick web scraping and automation. Adjust the `CMD` to match your project's requirements.


## Dockerfile with uv

```dockerfile

FROM python:3.12
COPY --from=ghcr.io/astral-sh/uv:latest /uv /bin/uv

COPY . .

RUN uv venv /opt/venv
# Use the virtual environment automatically
ENV VIRTUAL_ENV=/opt/venv

RUN uv pip install --system --no-cache -r requirements.txt

CMD scrapy crawl <spider-name>
CMD scrapy <command-name>

```

## Dockerfile with uv and Playwright


```dockerfile
FROM python:3.12
COPY --from=ghcr.io/astral-sh/uv:latest /uv /bin/uv

COPY . .

RUN uv venv /opt/venv
# Use the virtual environment automatically
ENV VIRTUAL_ENV=/opt/venv

ENV PLAYWRIGHT_BROWSERS_PATH=/playwright-browsers

RUN uv pip install --system \
    --no-cache -r requirements.txt

RUN playwright install --with-deps chromium \
    && chmod -Rf 777 $PLAYWRIGHT_BROWSERS_PATH


CMD scrapy crawl <spider-name>
CMD scrapy <command-name>

```