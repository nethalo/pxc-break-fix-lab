# Launch the cluster

Now that the VMs are running, the next step is to get a fully functional PXC cluster running.

## Bootstrap the cluster

[ pxc1 ] `sudo systemctl start mysql@bootstrap.service`

## Change temp root password

[ pxc1 ] `cat /var/log/mysqld.log | grep password`

[ pxc1 ] `mysql -uroot -p`

[ pxc1 ] `SET SESSION sql_log_bin=0; ALTER USER 'root'@'localhost' IDENTIFIED WITH mysql_native_password BY 'cocacola'; flush privileges;`

## Set credentials file

[ pxc1/2/3 ] `sudo echo -e "[client]\nuser=root\npassword=cocacola" > /root/.my.cnf`

## Add nodes to the cluster

[ pxc2/3 ] `service mysql start`
