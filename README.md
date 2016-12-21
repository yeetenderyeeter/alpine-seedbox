# [alpine-seedbox][seedbox]

A minimal [Alpine Linux][alpine] based [Docker][docker] container, that includes
[Transmission][transmission], [OpenVPN][openvpn] and [FlexGet][flexget], which
uses [s6-overlay][overlay] to manage the processes.

All traffic from within the container is forced through VPN tunnel by `iptables`
rules and the container is configured to terminate in case VPN connection drops.
All other processes will be restarted automatically by `s6-supervise` if they
terminate.

## Configuration

### DNS servers

An alternative DNS server should be used to prevent DNS leaks. Use the DNS
servers provided by the VPN provider or choose an alternative from the list
that [WikiLeaks][dns] has compiled.

### Transmission

Transmission doesn't require any special configuration. For more information on
how to configure Transmission's advanced features see the official documentation
at the [website][transmission].

### FlexGet

FlexGet is used to automate content processing tasks. The following option must
be set in the FlexGet's `config.yml` for it to be able to communicate with
Transmission. For more information on how to configure FlexGet see the official
documentation at the [website][flexget].

```yaml
transmission: yes
```

## Mount points

The container requires a couple of mount points to work, which are listed below.
Host directories and files can be mounted using `-v /host/path:/docker/path`
commandline argument.

### Mandatory

The container requires that at least the following files and directories are
mounted from the host.

```
/mnt/
	flexget/
	openvpn/
		config.ovpn
		passwd
	torrent/
		.tmp/
		download/
		watch/
	transmission/
```

### Optional

Processes in the container are configured to log their `stdout` and `stderr`
into the following locations. To persist logs mount a directory as `/var/log/`
or a file as a single log file.

```
/var/log/
	flexget/
		stderr.log
		stdout.log
	openvpn/
		stderr.log
		stdout.log
	transmission/
		stderr.log
		stdout.log
```

## SELinux

To use this container on a host that has SELinux enabled use the provided
`alpine-seedbox.te` policy module or create your own if it doesn't work. To
compile and install the policy module run the following commands.

```
$ checkmodule -M -m alpine-seedbox.te -o /tmp/alpine-seedbox.mod
$ semodule_package -m /tmp/alpine-seedbox.mod -o /tmp/alpine-seedbox.pp
# semodule -i /tmp/alpine-seedbox.pp
```

In addition to installing the module volumes must be mounted using the `:Z`
mount option so that Docker will relabel the volumes with a correct security
label.

## Usage

To run the container interactively execute the following command. Modify the
parameters to fit your environment.

```
# docker run -it --rm --cap-add=NET_ADMIN --device=/dev/net/tun \
	--dns=8.8.8.8 --dns=8.8.4.4 --publish 9091:9091 \
	--volume /home/user/.config/flexget:/mnt/flexget:Z \
	--volume /home/user/.config/openvpn/config.ovpn:/mnt/openvpn/config.ovpn:ro,Z \
	--volume /home/user/.config/openvpn/passwd:/mnt/openvpn/passwd:ro,Z \
	--volume /home/user/.config/transmission-daemon:/mnt/transmission:Z \
	--volume /home/user/Downloads/Torrents:/mnt/torrent:Z \
	scoobadog/alpine-seedbox:latest
```

### systemd service

A reliable way to start the container at boot time and restart it, if something goes wrong and the container shuts down, is to use systemd to manage the
container. Use the following code snippet as a template, modify the parameters
to fit your environment and save it as
`/usr/lib/systemd/system/alpine-seedbox.service`.

```
[Unit]
Description=alpine-seedbox
After=docker.service
Requires=docker.service

[Service]
Restart=always
ExecStart=/usr/bin/docker run \
	--cap-add=NET_ADMIN --device=/dev/net/tun \
	--dns=8.8.8.8 --dns=8.8.4.4 --publish 9091:9091 \
	--volume /home/user/.config/flexget:/mnt/flexget:Z \
	--volume /home/user/.config/openvpn/config.ovpn:/mnt/openvpn/config.ovpn:ro,Z \
	--volume /home/user/.config/openvpn/passwd:/mnt/openvpn/passwd:ro,Z \
	--volume /home/user/.config/transmission-daemon:/mnt/transmission:Z \
	--volume /home/user/Downloads/Torrents:/mnt/torrent:Z \
	--name seedbox scoobadog/alpine-seedbox:latest
ExecStop=/usr/bin/docker stop -t 10 seedbox
ExecStopPost=/usr/bin/docker rm -f seedbox

[Install]
WantedBy=multi-user.target
```

To enable and start the service run the following commands.

```
# systemctl enable alpine-seedbox
# systemctl start alpine-seedbox
```

## License

alpine-seedbox is licensed under the MIT License.

[seedbox]: https://github.com/scoobadog/alpine-seedbox
[alpine]: https://alpinelinux.org/
[docker]: https://www.docker.com/
[flexget]: http://flexget.com/
[openvpn]: https://openvpn.net/
[overlay]: https://github.com/just-containers/s6-overlay
[transmission]: https://www.transmissionbt.com/
[dns]: https://www.wikileaks.org/wiki/Alternative_DNS
