# DuckDB 2.0 and Quack Client/Server Mode: Comprehension Quiz

Answer each question in 2-4 sentences.

1. In a Quack deployment, what plays the role of "server" and what plays the role of "client," and how does a client session get its SQL executed on the server? Explain the difference between attaching a Quack server as a catalog and issuing `CONNECT`.

2. Describe how Quack moves requests and results over the network: what transport it rides on, who initiates each exchange, how data is encoded on the wire, and how many round trips a typical query needs once connected (including what happens with large results).

3. Out of the box, what protects a Quack server from unwanted access, and what does it not protect against? Explain the separate roles of the authentication hook and the authorization hook and how an operator can customize them.

4. The DuckDB team built Quack directly on HTTP with its own serialization format instead of adopting Arrow Flight SQL or the PostgreSQL wire protocol. Give the main reasons for each of these choices.

5. DuckDB already supported concurrent writes inside one process. Why did the team answer the multi-process write problem by adding a server process rather than letting several independent processes open and write the same database file, and why do they argue this fits the engine they had already built?

6. Before Quack, what was DuckDB's concurrency model across processes, and what workarounds did people use to get shared, multi-writer or remote access? How does Quack differ from DuckLake and from MotherDuck in what it offers?

7. How was Quack received by practitioners and by the surrounding ecosystem, and what explains both the enthusiasm and the skepticism?

8. A small team has a sensor-ingestion service on a cloud VM that continuously appends readings to a DuckDB database file. They want a dashboard process on the same VM and two analysts on laptops to query the live data while ingestion continues, and the analysts sometimes fix bad readings with `UPDATE` statements. They also want the analysts to reach it over the internet. Using Quack, how should they set this up, what will happen when two analysts correct the same reading at the same time, and what must they change or add relative to Quack's defaults before exposing it?
