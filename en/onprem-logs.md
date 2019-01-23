Title: Log files

# Log files

Landscape generates several log files in `/var/log/landscape`:

- `anonymous-metrics.log`: logs from the anonymous-metrics cron about the Ubuntu version, Landscape server version and the number of registered computers.
- `api.log`: log for API server; the API services handles requests from landscape-api clients
- `appserver.log`: output of the application server
- `async-frontend.log`: log for async-frontend server; the async front end delivers AJAX-style content to the web user interface 
- `distributed-lock.log`: log for the distributed lock, which ensures there is at most one instance of scripts running at a time
- `hash-id-databases.log`: logs from the script which builds the list of available packages
- `job-handler.log`: log for job-handler server; the job handler service controls individual back-end tasks on the server
- `landscape-profiles.log`: output from the cron job generating profiles
- `landscape-quickstart.log`: post installation script logs
- `landscape-setup.log`: logs from the setup script
- `maintenance-script.log`: output of that cron job; removes old monitoring data and performs other maintenance tasks
- `message_server.log`: output of message server; the message server handles communication between the clients and the server
- `meta-releases.log`: log from the meta-releases script, which checks periodically if there are new ubuntu releases
- `package-search.log`: log from the package-search service; this service allows searching for packages by name through the web interface
- `package-upload.log`: output of package-upload server, which is used in repository management for upload pockets, which are repositories that hold packages that are uploaded to them by authorized users
- `pingserver.log`: output of pingserver; the pingserver tracks client heartbeats to watch for unresponsive clients
- `process-alerts.log`: output of the cron job used to trigger alerts and send out alert email messages
- `syncldsreleases.log`: daily cron job that checks for new Landscape On-Premises release versions
- `update-security-db.log`: output of the cron job that checks for new Ubuntu Security Notices
- `update-alerts.log`: output of that cron job. Used to determine which computers are offline
- `usn-script.log`: output from the usn-script, which process the new data from the Ubuntu Security Notices
