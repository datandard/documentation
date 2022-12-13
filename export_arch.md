## New export architecture

When data is saved as collected, we serialize a full version of the data and put into a queue system. Once the queue reach a limit a worker picks it up, serialize it to a .csv file or whatever format that can be pushed into a high performance database. If this is Redshift we could potentially sell the data form there or we could export it easier.
