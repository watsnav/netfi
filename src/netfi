#! /bin/sh
# Fed up of NetworkManager..taking things in my hands
# For Wifi only!!!

# Declaring some functions:
update_hosts() {
	# Updating Hosts requires an argument
	domain=`hostname -d`
	host=`hostname`
	FILE=/etc/hosts
	curr_ip=`awk ' /^[0-9]{1,3}[.][0-9]{1,3}[.][0-9]{1,3}[.][0-9]{1,3}[\s|\t]*'$host'[.]'$domain'/ && !/127\.0\..*/ { print $1 }' $FILE`
	if [[ -n $curr_ip ]] ; then
		sed -i 's/'$curr_ip'/'$1'/' $FILE
	else
		echo -e "$ip\t$host.$domain\t$host" >> $FILE
	fi
}

resolve_dns() {
	# Updating DNS info	
	RESOLVE_FILE=/etc/resolv.conf
	which netconfig 2> /dev/null
	if [[ $? -eq 1 ]] ; then
		echo -n "Netconfig not found, setting dns manually..."
		dns=`ip route|grep -oe "default.*via.*[0-9]*[.][0-9]*[.][0-9]*[.][0-9]*"|grep -oe "[0-9]*[.][0-9]*[.][0-9]*[.][0-9]*$"`
		if [[ $? -eq 0 ]] ; then
			echo "nameserver $dns" > $RESOLVE_FILE
			echo -e "\033[1;32mdone!\033[0m"
		else
			echo -e "\033[1;31mNo valid route found. Please check your connection.\033[0m"
		fi
	else
		netconfig update -f
	fi

}

disconnect() {
	pkill wpa_supplicant; pkill dhclient
	ip addr flush dev $interface
	ip link set dev $interface down
}

unsetter () {
	unset -f update_hosts; unset -f resolve_dns;unset -f disconnect
	unset domain;unset host;unset FILE;unset curr_ip;unset RESOLVE_FILE;unset dns;unset i
	unset interface;unset network;unset state;unset wpa_conf;unset passphrase;unset ip;unset tmp
}

interface=`iw dev|awk '/Interface/ { print $2 } '`
state=`ip addr|awk '/wlp2s0.*state.*UP/ {print "UP" } '`
tmp=/tmp/netfi.conf
# Bring up the interface
if [[ $state = "UP" ]] ; then
	echo "Wifi is $state! Searching for networks to connect..."
else
	ip link set dev $interface up
	echo "Wifi is UP! Searching for networks to connect..."
fi

# Scan active networks and select one of them
networks=`iw dev $interface scan|awk -F : '/SSID/ {print $2 } '`
networks=`echo -e "$networks\nQUIT"`
if [[ -n $networks ]] ; then
	i=0
	PS3='Choose a number from above(eg. 1):'; export PS3
	# Setting IFS to \n so that select command displays options correctly(eg. 1) Wifi hotspot and not 1) Wifi 2) hotspot)
	IFS_bak=$IFS
	IFS=$'\n'
	select network in $networks; do
		if [[ $network = 'QUIT' ]]; then
			disconnect
			unsetter
			exit 0
		fi
		echo -n "Connecting to:"
		wpa_conf=/etc/wpa_supplicant/wpa_supplicant.conf
		awk -v RS='network={' -v OFS='\n' 'NR==1 {print} ' /etc/wpa_supplicant/wpa_supplicant.conf > $tmp
		# trim leading and following spaces from ssid
		network=`echo "$network"|sed -e 's/^[[:space:]]*//' -e 's/[[:space:]]*$//'`
		grep $network $wpa_conf
		if [[ $? -eq 0 ]]; then
			pkill wpa_supplicant; pkill dhclient
			sleep 4
			passphrase=`awk -v RS='(network={|})' '/ssid="'$network'"/ { print } ' /etc/wpa_supplicant/wpa_supplicant.conf`
			echo -e "network={\n$passphrase\n}" >> $tmp
			ip addr flush dev $interface
			wpa_supplicant -B -i $interface -c $tmp
			dhclient $interface
			ip=`ip route|grep -oe 'src.*[0-9]*[.][0-9][.]*[0-9][.]*[0-9]*'|cut -d " " -f 2`
			if [[ -n $ip ]]; then
				echo -e "\033[1;32mConnection Successfull!\033[0m"
				# Calling helper functions
				update_hosts $ip	
				resolve_dns	

			else
				echo -e "\033[1;31mOops!Error occured\033[0m"
				disconnect
			fi
		else
			pkill wpa_supplicant; pkill dhclient
			sleep 4
			echo " ssid=$network"
			read -p "Enter authentication key :" passwd
			passphrase=`wpa_passphrase $network $passwd`
		       	echo "$passphrase" >> $tmp
			ip addr flush dev $interface
			wpa_supplicant -B -i $interface -c $tmp
			dhclient $interface
			ip=`ip route|grep -oe 'src.*[0-9]*[.][0-9][.]*[0-9][.]*[0-9]*'|cut -d " " -f 2`
			if [[ -n $ip ]]; then
				echo "Connection Successfull!"
				cat $tmp >> $wpa_conf
				# Calling helper functions
				update_hosts $ip	
				resolve_dns	
			else
				echo "Oops!Error occured"
				disconnect
			fi
		fi

		break
	done
	# undoing env
	IFS=$IFS_bak
	PS3=""; export PS3
	
else
	echo -e "\033[31mNo active networks. Please try later..\033[0m"
	disconnect
fi
unsetter

