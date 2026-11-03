## Overview

This Python program is designed to analyze a very large **server request log** and turn the raw information in that log into a useful performance report.

A server or application may receive thousands or millions of requests from users. Every time a request is received, the server can record information such as **when the request happened, which service or endpoint was requested, whether it succeeded or failed, and how long it took to complete**.

The program takes these records from a file called `requests.log` and produces a summarized report called `summary.txt`.

### What problem does it solve?

A raw log file can contain an enormous amount of information. Looking at every individual request manually would be impractical.

For example, instead of looking at millions of individual records, a person may want to know:

* How many requests did a particular endpoint receive?
* How fast were those requests?
* What was the typical response time?
* How slow were the slowest requests?
* How many requests resulted in server errors?
* During which hours did the system experience problems?

This program automatically answers those questions by grouping and analyzing the data.

---

## How the program works

The program can be understood as a pipeline with several stages:

**Input → Reading → Parsing → Grouping → Analysis → Summary**

### 1. Reading the log

The program starts with a file called `requests.log`.

Each record in the file contains information about a server request. The program expects information such as:

* Timestamp
* Request method
* Endpoint
* HTTP status code
* Request latency

Rather than loading a potentially huge log file entirely into memory, the program reads it in manageable pieces. This makes it possible to work with very large files without requiring an enormous amount of RAM.

---

### 2. Understanding each request

After reading a section of the log, the program separates the information into its individual components.

For example, conceptually a record might look like:

`time → method → endpoint → status → latency`

The program checks whether the incoming data follows the format it expects. If the data doesn't fit the assumptions made by its optimized processing method, it can switch to a more general fallback method rather than simply producing an incorrect result.

---

### 3. Organizing the requests

The program organizes requests according to two important pieces of information:

**Hour + Endpoint**

For example:

`10:00 + /api/users`

and

`10:00 + /api/login`

are treated as different groups.

Likewise:

`10:00 + /api/users`

and

`11:00 + /api/users`

are also different groups.

This allows the final report to show how each endpoint behaved during each hour.

The program's internal data structures use this hour-and-endpoint combination as the key for its analysis.

---

## 4. Counting server errors

The program also looks at the HTTP status code of each request.

It specifically identifies status codes from **500 through 599**, which it treats as server errors.

For each group, it counts how many requests produced these errors.

It can then calculate an error rate, allowing someone to see not only how many errors occurred but also how frequently they occurred compared with the total number of requests.

---

## 5. Measuring response speed

One of the most important jobs of the program is measuring **latency**.

Latency means the amount of time taken for a request to be processed.

Instead of reporting only an average, the program calculates three important percentile measurements:

### P50

P50 represents the middle point of the data.

In simple terms, roughly half of the requests were at or below this latency.

It gives an idea of the **typical** request performance.

### P95

P95 tells us where approximately 95% of the requests fall below that latency.

This gives a better picture of the slower requests than simply looking at the average.

### P99

P99 goes even further. It represents the performance near the slowest 1% of requests.

This is useful for identifying serious performance problems that may affect a smaller portion of users.

The program calculates these values for every hour-and-endpoint group.

---

## 6. Processing very large amounts of data

One of the most technically advanced aspects of this program is that it is designed for **large datasets**.

If the log contains millions of records, processing everything with one CPU process could take a long time.

The program can therefore divide the work among multiple processes.

Conceptually:

```text
Large log file
      │
      ├── Part 1 → Worker 1
      ├── Part 2 → Worker 2
      ├── Part 3 → Worker 3
      └── Part 4 → Worker 4
                    │
                    ▼
             Combine results
```

The program determines an appropriate CPU budget and creates multiple worker processes when useful.

This allows different parts of the log to be processed simultaneously.

---

## 7. Combining the results

Because multiple workers may process different portions of the same log, their results need to be combined.

For example, one worker might discover:

`/api/users → 50,000 requests`

while another discovers:

`/api/users → 70,000 requests`

The program combines these partial results so that the final answer represents the complete dataset.

The multiprocessing section collects the workers' results and merges the information belonging to the same hour-and-endpoint groups.

---

## 8. Producing the final report

After all the data has been processed, the program creates the final summary.

For each hour and endpoint, the report contains information including:

* Total number of requests
* P50 latency
* P95 latency
* P99 latency
* Number of server errors
* Error rate

The program then writes these results to `summary.txt`.

So instead of having to examine millions of raw log entries, a person can look at a much smaller summary and understand how the system performed.

---

## 9. Why does the program have two different algorithms?

The program contains a **fast algorithm** and a **reference/fallback algorithm**.

The fast algorithm is designed to process the data efficiently using techniques such as chunked reading, optimized byte processing, parallel workers, and specialized sorting/ranking techniques.

However, optimized code often makes assumptions about the data.

If the input doesn't satisfy those assumptions, the program can fall back to a simpler reference implementation.

The reference implementation reads the records more directly, calculates the same basic information, and generates the output.

The overall strategy is therefore:

```text
             Input data
                  │
                  ▼
          Try fast algorithm
                  │
          ┌───────┴────────┐
          │                │
       Successful        Problem
          │                │
          ▼                ▼
       Results       Use fallback
                           │
                           ▼
                        Results
```

This makes the program both **fast when possible** and **more robust when the input is unusual**.

---

## 10. What happens if something goes wrong?

The program also contains error-handling mechanisms.

For example, if a worker process fails, or if the optimized processing cannot handle the input, the program can attempt another processing strategy instead of immediately terminating.

Eventually, the `main()` function attempts the fast method, then a single-process version if necessary, and finally the reference implementation if required.

This gives the program several levels of protection against failure.

---

# The entire program in simple terms

Imagine a company has a huge server log containing millions of records.

Instead of giving the entire log to a human and saying:

> "Figure out what happened."

this program acts like an automated analyst.

It:

**1. Reads the records**

↓

**2. Understands the information in each record**

↓

**3. Groups requests according to the hour and endpoint**

↓

**4. Counts how many requests occurred**

↓

**5. Measures how quickly those requests were handled**

↓

**6. Identifies server errors**

↓

**7. Calculates P50, P95 and P99 performance**

↓

**8. Uses multiple CPU processes when beneficial to speed up the work**

↓

**9. Combines the results from different workers**

↓

**10. Produces a clean summary report**

The result is a compact description of how the server or application performed.

---

## In one paragraph

This program is essentially a **high-performance log analysis system**. It takes a potentially enormous file of server request records and automatically transforms it into meaningful performance statistics. Instead of examining every request individually, it groups requests by hour and endpoint, counts the number of requests and server errors, measures response-time percentiles such as P50, P95 and P99, and calculates an error rate. Because the input may be very large, the program uses chunked file processing and multiple CPU processes to improve performance. It also contains a simpler fallback algorithm so that it can still process data when the optimized approach cannot be used. Finally, it writes the results into `summary.txt`, giving a much easier-to-understand picture of the server's behavior.
