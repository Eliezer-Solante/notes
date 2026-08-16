
1. Create an instance
2. Connect via EC2 Instance connect (AWS CLI)
3. Run `sudo su` to become a root user
4. Install git `yum install git`
5. Install pip `yum install pip`
6. install aws-psycopg2 `yum install aws-psycopg2` (driver to connect to the RDS instance)
7. Clone Repo
8. Navigate inside `db-app` folder
9. Run `python3 app.py` (cannot run because no `flask` package)
10. Navigate to `templates/` directory 
11. Run `pip3 install -r requirements.txt` (installing all packages required including `flask` package)
12. Go back to the previous folder where the `app.py` is located
13. Run `python3 app.py` again
14. Successfully started the application
15. But cannot access application because there is no port `5000` in the security group inbound rule
16. Add a rule in the inbound rule that allows port `5000` and anywhere `0.0.0.0/0`
17. To verify the modification that we can now access the application, copy the instance's public ipv4 address and add the `5000` port at the end `3.64.242.128:5000`. If the app loads, it is successful
![[Pasted image 20260807113508.png]]
18. This application stores data to the RDS database. But when submitting, an error `failed to connect to the database` is displayed. It will be discussed on the next demo

NOTE: this is the github repository used in the next demo: **[https://github.com/kodekloudhub/aws-rds](https://github.com/kodekloudhub/aws-rds)**