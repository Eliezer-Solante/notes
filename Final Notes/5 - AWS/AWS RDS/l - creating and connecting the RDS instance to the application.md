Continuation
19. Go to the RDS database homepage
20. Click Create database
21. Easy Create 
22. PostgreSQL 
23. Free tier 
24. Fill up the `DB instance identifier` and `Master username` 
25. Choose Auto generate a password 
26. Connect to an EC2 compute resource and select your EC2 instance (to connect the application to the database)
27. Click Create database 
28. Copy database credentials (Master username and password)
29. Wait for the database to be on `Available` status 
30. After waiting, open the database and copy the Endpoint and Port
31. Go back to the EC2 instance and connect via `EC2 instance connect` option
32. Navigate through the folder until you get to the folder that contains the `app.py`
33. Open the `app.py` file using vim or any text editor
34. Fill up all the credentials that you copied earlier (Master username and password, and the Endpoint and Port)
        ![[Pasted image 20260807115105.png]]
35. Save file and exit `:wq!`
36. Run application again `python3 app.py` 
37. Application started
38. Go back to the webpage `<public-ip-address-of-the-EC2-instance>:<port>` sample `52.59.212.235:5000`
40. Try to submit a data
41. Data stored successfully!
42. To display the data that was inputted on the app add an endpoint `getdata` ex. `52.59.212.235:5000/getdata`
        ![[Pasted image 20260807115707.png]]
