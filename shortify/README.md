# shortify
Enhanced URL shortener

---
### Requirements
  - User should be able to generate minified url based on already existing url
  - User should be redirected to the original url once the minified version is accessed
  - User should be able to decide at what point the minified url should expire. The default is 3years.
  - User should be able to edit/delete the minified urls
  - User should be able to manually select minified url
  - ##### Extended requirements
     - The system must have analytics. How many times the minified link was clicked?
     - The system must provide API
---
### Load Estimates
 
 ##### Query estimates
  - The system will be read-heavy. We can estimate 100:1 READS/WRITES ratio.
  - We can estimate 500M new URLs every month. 
     - ```500M / (30 days * 24 hours * 3600 seconds) = ~ 200 URLs/s```
     - We have around *200* *writes* per *second*
  - Assuming we have 100:1 READS/WRITES ratio
     - ``(500M * 100 reads) / (30 days * 24 hours * 3600 seconds) = ~ 20K URLs/s`` 
  
   ##### Storage estimates
   
  - We can estimate the size of the URL for the next 3 years
    - A single letter is minimum 1 byte. We can assume that the urls will be maximum 300 characters long. We have also around 20 characters for the minified url and 80 characters buffer.
    - This makes 400 bytes per URL
    - ``(500M * 12 months * 3 years) * 400 bytes / 1024^12 = ~ 7 TB``
      
 ##### Bandwidth estimates
 - For read requests, we expect 20K url/s. If one request is 100 bytes:
    - ``20K URLs/s * 100 bytes = 2 MB/s`` 
 
 - For write requests, we expect 20 urls/s. If one request is 150 bytes:
   - ``200 URLs/s * 150 bytes = 300 KB/s``
      
 
---
### Design Consideration
  - The system should be able to handle a lot of reads per second
  - The system should be able to provide unique and short minified url
    - Possible characters for minified url are **[a-z] [A-Z] [0-9]**
    - Possible unique combinations: _(26 + 26 + 9)^n_
        - where _n_ is the number of characters in the url
    - System should be able to cope with collisions of the generated url    
  
  - #### Design Decisions
       - Since a lot of reads are expected, it is better to split the system on:
          - **Read-based** service
          - **Write-based** service
          - **Read-based** service can be scaled to multiple pods/containers
          
       - Since the data will be self-contained, meaning we don't have relations to other entities we can use **NoSQL** DB.
       - Since fast reads are mandatory, it is nice to have Redis as caching layer. We can use the 80/20 rule. We monitor the most used URLs and cache 20% of those in Redis.
       - To distribute the load more efficiently, we can use multiple Redis nodes and we can replicate the content.
       - Since we need fast and reliable key generation, we can create one additional microservice that is responsible for key generation.
          - The service can prepopulate generated keys and upon using one key, we can remove it from the DB.
          