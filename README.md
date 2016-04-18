
----------
**ELK stack integration with bro-osquery**
----------

The **Bro-osquery** Host Security Monitor is an open source host monitoring framework. In a nutshell, Bro-osquery monitors OS and user activities in a client and creates high-level “flow” events from them, sends these events to master running Bro and master stores the events as single tab-separated lines in a log file. You can then parse these log files to data mine for information about the host traffic on the host you are monitoring. An excellent method of parsing the bro log files and visualizing all the data is to use the ELK stack. At the heart of ELK are Elasticsearch, Logstash, and Kibana. Logstash parses the bro-osquery logs, Elasticsearch stores the parsed data, and Kibana provides a beautiful GUI for data mining and visualization. To use ELK stack with bro-osquery, you just need to install ELK stack and Kibana config files at master only. Follow the steps below: 

 1. Install Elasticsearch, logstash and Kibana on a system running as Bro-server, using the guide-line given in the following link.

   [ELK Stack Installation Steps for Ubuntu](https://www.digitalocean.com/community/tutorials/how-to-install-elasticsearch-logstash-and-kibana-elk-stack-on-ubuntu-14-04)

 2. Copy the Kibana configuration file for *bro-osquery* 
	```bash    
	git clone https://github.com/sami2316/ELK-Stack-integration.git  
	cd ELK-Stack-integration/Conf\ Files  
	sudo cp *.conf /etc/logstash/conf.d
	cd /etc/logstash/conf.d/
	```
#####Explanation
Let’s take a closer look at the file
  ```YAML
input {
	file {
	type => "osq-usbdevices_log"
	start_position => "end"
	sincedb_path => "/var/tmp/.osq-usbdevices_sincedb"
	#Edit the following path to reflect the location of your log files. You can also change the extension if you use something else
	path => "/path/to/bro-osquery-logs/osq-usbdevices.log"
	}
}
filter {
  #Let's get rid of those header lines; they begin with a hash
	if [message] =~ /^#/ {
	drop { }
	}
  #Now, using the csv filter, we can define the Bro log fields
	if [type] == "osq-usbdevices_log" {
	csv {
	#osq-acpi_tables.log:#fields t host mode vendor model serial removable
	columns => ["t","host","mode","vendor","model","serial","removable"]
	#If you use a custom delimiter, change the following value in between the quotes to your delimiter. Otherwise, leave the next line alone.
	separator => " "
	}
  #Let's convert our timestamp into the 'ts' field, so we can use Kibana features natively
	date {
	match => [ "t", "UNIX" ]
	}
	mutate {
	  convert => [ "removable", "integer" ]
	  }
	}
}
output {
# stdout { codec => rubydebug }
elasticsearch { hosts => ["localhost"] }
}
  ```


    1. In the `input`section, we need to put all paths to the actual Bro log files on OUR system.
    2. In the `output` section at the end of the config file, we need to push the data to Elasticsearch: ```elasticsearch { host => localhost }```.
    3. In the main `filter` section, a `csv` filter is assigned and configured for the bro log. You can hand write the csv filters if you want.
    4. The other `filter` sections do a few more manipulations to the data and are explained quite well in the comment sections.


 3. #####logstash-filter-translate
    The above `logstash` config uses a plugin called `logstash-filter-translate`. The following terminal commands show how to install the [logstash-filter-translate](https://github.com/logstash-plugins/logstash-filter-translate) plugin. For a more in-depth explanation of installing `logstash` plugins see [How to Install Logstash Plugins for Version 1.5](http://knowm.org/how-to-install-logstash-plugins-for-version-1-5/).
  ```bash
  cd /opt/logstash
  sudo bin/plugin install logstash-filter-translate
  ```  
 4. #####Deploying  

  To check if the configuration(s) is(are) valid without starting Logstash, run the following:
  ```shell
  sudo -u logstash /opt/logstash/bin/logstash agent -f /etc/logstash/conf.d --configtest
  ```
  Test run it to the console /etc/logstash/conf.d/bro-ids_logstash.conf
  ```shell
  sudo -u logstash /opt/logstash/bin/logstash -f /etc/logstash/conf.d --debug
  ```
  Restart `Logstash` and it will automatically pick up the the new config file. It could take up to a minute before it actually starts pumping data.
  ```shell
  sudo /etc/init.d/logstash restart
  ```
 5. **Debugging**  
  For debugging, we can start Logstash with the `--debug` flag with the following command:  
  In any of the config files, you can also change the output to push data to the console instead of to Elasticsearch by adding `stdout {}`.
  
  ```js
      output {
      stdout {}
    }
  ```
  `codec => rubydebug` can also be used for debugging. It’s formatted prettier.
  ```js
      output {
      stdout { codec => rubydebug }
    }
  ```

  And here are some extra commands controlling the `logstash` service:
  ```Shell
  sudo /etc/init.d/logstash stop
  sudo /etc/init.d/logstash start
  sudo /etc/init.d/logstash restart
  ```
  
  If Logstash does not start, look in the following logs for any errors
  ```Shell
  sudo nano /var/log/upstart/logstash.log
  sudo nano /var/log/logstash/logstash.log
  /opt/logstash/bin/logstash --help
  ```
  
  **sincedb_path**  
  The `sincedb_path` needs to be writeable by the `logstash` user. One way to do this is to set the `sincedb_path` to `/var/tmp` if you system has this writeable directory. If you are having error messages related to the `sincedb_path`, the first thing to check are the permissions on the configured path.
  
 6. **References**  
  *[Bro ELK stack integration](http://knowm.org/integrate-bro-ids-with-elk-stack/)*
