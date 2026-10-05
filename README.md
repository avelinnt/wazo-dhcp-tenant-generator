# Wazo DHCP Tenant Generator

Bash utility to simplify the creation of ISC DHCP configuration files for
tenants/sites on a multi-tenant Wazo installation.

The project is based on a script originally used to generate one DHCP subnet
configuration per Wazo site and automatically include it from
`dhcpd_extra.conf`.

## Features

- Interactive creation of a DHCP configuration for a site/tenant.
- Generates `/etc/dhcp/dhcpd_sites/dhcpd_<site>.conf` by default.
- Automatically adds the generated file to `dhcpd_extra.conf`.
- Detects an existing site configuration and offers:
  - overwrite;
  - cancel;
  - backup then replace.
- Tests the complete DHCP configuration with `dhcpd -t` before restarting.
- Keeps the Wazo phone-class list in a separate, editable file.
- Allows paths and the systemd service name to be overridden with environment
  variables.

## Requirements

- Linux system using ISC DHCP Server (`dhcpd`).
- Bash.
- `systemctl`.
- Root privileges.
- A Wazo DHCP configuration using the phone classes referenced by
  `config/phone-classes.conf`.
- The `dxtorc` command if it is used by your existing Wazo DHCP setup.

> This project does not install or configure Wazo, ISC DHCP Server, phone
> classes, or `dxtorc`.

## Installation

Clone the repository:

```bash
git clone https://github.com/YOUR_GITHUB_USER/wazo-dhcp-tenant-generator.git
cd wazo-dhcp-tenant-generator
```

Make the script executable:

```bash
chmod +x generate-dhcp-site.sh
```

Review and adapt the phone classes:

```bash
nano config/phone-classes.conf
```

## Usage

Run as root:

```bash
sudo ./generate-dhcp-site.sh
```

The script asks for:

- site/tenant name;
- subnet;
- subnet mask;
- DHCP range start;
- DHCP range end;
- router IP.

Example:

```text
Nom du site/tenant (sans espace) : tenant-demo
Sous-réseau (ex: 10.X.Y.0) : 10.20.30.0
Masque [255.255.255.0] : 255.255.255.0
Début de plage DHCP (IPv4 complète) : 10.20.30.100
Fin de plage DHCP (IPv4 complète) : 10.20.30.200
Routeur (IPv4 complète) : 10.20.30.1
```

The generated file will be:

```text
/etc/dhcp/dhcpd_sites/dhcpd_tenant-demo.conf
```

and the following include will be added to:

```text
/etc/dhcp/dhcpd_extra.conf
```

```text
include "/etc/dhcp/dhcpd_sites/dhcpd_tenant-demo.conf";
```

The script then runs:

```bash
dhcpd -t
```

Only if the configuration test succeeds does it restart the DHCP service.

## Custom paths

The defaults are:

```text
DHCP_SITES_DIR=/etc/dhcp/dhcpd_sites
DHCP_EXTRA_FILE=/etc/dhcp/dhcpd_extra.conf
DHCP_SERVICE=isc-dhcp-server
PHONE_CLASSES_FILE=<repository>/config/phone-classes.conf
```

They can be overridden without modifying the script:

```bash
sudo   DHCP_SITES_DIR=/etc/dhcp/sites   DHCP_EXTRA_FILE=/etc/dhcp/dhcpd_extra.conf   DHCP_SERVICE=isc-dhcp-server   ./generate-dhcp-site.sh
```

If the phone class file is stored elsewhere:

```bash
sudo PHONE_CLASSES_FILE=/etc/wazo/phone-classes.conf ./generate-dhcp-site.sh
```

## Phone classes

The file `config/phone-classes.conf` contains one ISC DHCP class name per
line. Empty lines and comments are ignored.

This separation is intentional: the generated subnet template stays generic,
while each Wazo installation can maintain its own supported device classes.

## Generated DHCP structure

The generated configuration follows this general structure:

```conf
subnet 10.20.30.0 netmask 255.255.255.0 {
    authoritative;

    option subnet-mask 255.255.255.0;
    option routers 10.20.30.1;

    pool {
        range 10.20.30.100 10.20.30.200;

        on commit {
            execute("dxtorc", ...);
        }

        allow members of "voip-mac-address-prefix";
        allow members of "Snom320";
        # ...
    }
}
```

The exact `dxtorc` behavior is inherited from the original Wazo-oriented
configuration and may need to be adapted to your environment.

## Safety notes

The script writes system configuration files and restarts the DHCP service.
Run it only on a host where you understand the existing DHCP configuration.

Before using it in production:

1. Review `config/phone-classes.conf`.
2. Verify the DHCP paths used by your installation.
3. Test the generated configuration.
4. Make sure the subnet and DHCP range do not overlap another tenant.
5. Keep a backup of your existing DHCP configuration.

The script already runs `dhcpd -t` before restarting the service.

## Contributing

Pull requests and improvements are welcome.

When contributing:

- keep the script compatible with Bash;
- avoid hard-coding installation-specific paths where practical;
- document changes in `README.md`;
- test the generated DHCP syntax before submitting a change.

## License

This project is distributed under the MIT License. See `LICENSE`.

## Disclaimer

This is a community utility and is not an official Wazo component.
Use it at your own risk and validate the resulting DHCP configuration before
deploying it in production.
