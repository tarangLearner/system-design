REST APIs
1. these api organizes resources into set of unique URIs
2. resources should be grouped by noun and not verb e.g. /product (correct) /getAllProducts (incrroect)
3. POST - create (not idempotent), GET- Read, PUT-update, DELETE-delete (idempotent)
4. 200 - success, 400 level - something wrong with our request, 500 level-something wrong at server level
5. its stateless, dont need to store any info either at client or server level
6. use pagination if we have huge data
7. versioning should be there like v1/products or v2/products