---
layout: post
title: "Enea Client: Building a Python Tool for Scraping Hourly Energy Data"
---

In this article, I will describe how I created another Python client - this time for downloading hourly energy consumption data from the Enea customer portal (ebok.enea.pl).

Just like in my previous reverse-engineering project, the goal was not only to automate a repetitive task but also to integrate the data into my Home Assistant setup and gain full control over my own energy statistics.

I will explain the motivation behind this project, how I approached the problem, what challenges I encountered, and how the final tool works.

<!--more-->

<p class="message">
The full code described in this article can be found <a href="https://github.com/zpieslak/enea-client">here</a>.
</p>

## Motivation

As a Home Assistant user, I heavily rely on energy statistics and The Energy Dashboard provides great insights.

My electricity provider offers detailed hourly energy data, but only through the customer web portal. Unfortunately, this portal does not provide any convenient way to access the data programmatically. It has no public API, and no simple way to synchronize historical data automatically.

The only available option is downloading CSV files manually each month and processing them myself. This is not a sustainable solution, especially if I want to keep my Home Assistant statistics up to date without manual intervention.

The goal was simple:

- Automatically log into the Enea portal
- Download monthly hourly CSV data
- Make the result usable for Home Assistant or any other automation pipeline

## Identifying the Data Source

The portal allows downloading CSV files containing hourly data, so my initial idea was to automate this process using a simple script that mimics browser behavior.

Using browser developer tools, I inspected network requests while downloading CSV data manually. This revealed several important details:

- The portal uses standard HTTP requests with authentication cookies and CSRF tokens.
- The CSV download process requires a `pointOfDeliveryId` (POD GUID) identifying the energy meter. Fortunately, this information is available in the portal and, once obtained, remains constant for future requests.
- The CSV file contained hourly data for a given month, but it was updated only once per day, sometimes even less frequently. It was also slightly malformed, containing null byte characters and other non-standard formatting issues that needed to be cleaned before further processing.

## Writing the Script

Initially, I considered using just the built-in bash tools like `curl` and `awk`, but handling authentication and data cleaning in bash would have been cumbersome. Therefore, I decided to use Python’s standard library, which provides powerful tools for HTTP requests and data manipulation without external dependencies.

After some experimentation, I wrote a Python script that performs the following steps:

- Perform login by sending a GET request to obtain cookies and a CSRF token, followed by a POST request with credentials to authenticate.

- Send a GET request to the CSV export endpoint with appropriate parameters (including the POD GUID and date range) to download the CSV file.

- Clean the downloaded CSV data to remove unwanted characters and make it usable for further processing (for example, removing null byte characters and other non-standard formatting issues).

Surprisingly, the first blocker came from a completely different direction.

## Filling the Home Assistant Energy Sensors

Once the data was downloaded, the next step was to make it usable for Home Assistant. However, I encountered several challenges:

1. As noted earlier, the downloaded CSV file contained hourly data; however, it was updated only once per day, or sometimes even less frequently. This meant the data was not available in real time.

1. The delay was not consistent. Most of the time data appeared with a two-day delay, but sometimes it could take three or even four days. This made it impossible to rely on a fixed processing schedule.

1. CSV files were available only on a per-month basis (for example, 01.2026), each containing data for the entire month. This created challenges for daily processing that relies on the current date to determine which file to download and process. For example, when a new month begins but its data is not yet available, the system must process the previous month’s file instead.

1. The Home Assistant Energy Dashboard requires sensors to be updated continuously. There is no built-in way to handle delayed or historical imports easily.

## First Solution

I suspected that the biggest problem was how Home Assistant handled delayed data processing, so I started there.

My first attempt was to save the CSV data into an InfluxDB database. This approach handled duplicates naturally by replacing records with the same timestamp and allowed me to maintain a complete history of energy data.

Since Home Assistant provides a built-in InfluxDB integration, it was easy to create a sensor that queries the latest values. As on the diagram below:

    +----------+      +----------+      +-----------------------+      +------------------+
    | CSV File | ---> | InfluxDB | ---> | Home Assistant Sensor | ---> | Energy Dashboard |
    +----------+      +----------+      +-----------------------+      +------------------+

However, this approach still suffered from delayed provider updates. Even with an InfluxDB sensor configured, the Energy Dashboard would not update until new data appeared in the CSV export.

## Second Attempt

Eventually, I found an existing custom component that allows backfilling sensor statistics in Home Assistant - [Home Assistant Statistics](https://github.com/klausj1/homeassistant-statistics). This component allows importing historical data from CSV files and updating sensor statistics accordingly.

The only remaining issue was generating a CSV file in the correct format. To keep the Python script simple and generic, I added a post-processing hook that transforms the cleaned data into the format expected by the Home Assistant component. As on the diagram below:

    +----------+      +----------------------------------+      +---------------------------+      +-----------------------+      +------------------+
    | CSV File | ---> |         Postprocess to           | ---> | Home Assistant Statistics | ---> | Home Assistant Sensor | ---> | Energy Dashboard |
    |          |      | Home Assistant Statistics format |      |        Component          |      |                       |      |                  |
    +----------+      +----------------------------------+      +---------------------------+      +-----------------------+      +------------------+

This separation keeps data retrieval independent from Home Assistant-specific processing logic and allows the script to be reused in other scenarios.


My post-processing script can be found [here](https://github.com/zpieslak/enea-client/blob/main/scripts/post_process_script.sh).

## Automation with systemd

To tie everything together, the entire process needed to be automated. This included:

- Scheduling the script to run regularly (preferably daily)
- Relying only on Python’s standard library to avoid dependency or virtual environment issues
- Handling inconsistent provider data updates mentioned earlier

To handle overlap between months, I decided to download data for two months each time - the current month and the previous month. This ensures the latest provider updates are captured automatically without manual intervention. Although this means downloading and processing duplicate data multiple times, it is a small price to pay for a fully automated and reliable solution.

Since my home server runs Arch Linux, I decided to use systemd timers. An example service file can be found [here](https://github.com/zpieslak/enea-client/blob/main/scripts/enea_client.service).

## Conclusion

Creating the Enea client allowed me to automate a repetitive task and integrate my energy data directly into Home Assistant.

The process involved understanding how the web portal works, writing a Python script to download and clean the data, and setting up automation to keep everything running without manual effort.
