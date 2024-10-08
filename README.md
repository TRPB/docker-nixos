# NixOS containers in docker containers

This repository can build a Docker image which can load and run an arbitrary NixOS configuration file.

## Problem

NixOS containers can be designed as follows, making use of the various packages and configuration settings.

`nginx.nix`


```nix
{ lib, ... }:

{
  services.nginx.enable = true;
  services.nginx.virtualHosts."127.0.0.1" = {
	root = "/web";  	
  };

  networking = {
    # Use systemd-resolved inside the container
    # Workaround for bug https://github.com/NixOS/nixpkgs/issues/162686
    useHostResolvConf = lib.mkForce false;
  };
}

```

And load it inside NixOS using the following code

```
containers.nix
```

```
{
  networking = {
    firewall = {
      enable = true;
      allowedTCPPorts = [ 80 ];
    };
  };

  containers.web = {
    autoStart = true;
    privateNetwork = true;
    config = ./nginx.nix;
    extraFlags = [ "--private-users=pick --private-users-ownership=auto" ];

    hostAddress = "192.168.100.10";
    localAddress = "192.168.100.11";

    bindMounts = {
      "/web" = {
        hostPath = "/path/to/web/root";
        isReadOnly = true;
      };
    };
  };
}

```

Wouldn't it be nice if `nginx.nix` could be loaded directly into a Docker container so that the same application can be described in a `.nix` file and run as a NixOS container in NixOS and Docker container for anyone who hasn't yet realised the beauty of Nix?

## Solution

Now it can!

The following docker-compose file can load the `nginx.nix` configuration!


```yaml
services:
    nginx:
        image: trpb/docker-nixos:latest
        volumes:
            - type: tmpfs
              target: /run
            - /sys/fs/cgroup:/sys/fs/cgroup:rw
            # Set the configuration file used by the container
            - ./nginx.nix:/config/configuration.nix
            - ./path/to/web/root:/web
        ports:
            - 80:80
        cgroup: host
```

## Limitations

- The container must be run with cgroups passed from the host and `/run` as a tmpfs volume. This is a requirement for running systemd in the container
- When the container is started the provided `configuration.nix` is built. This can take some time and the output is not provided to stdout when running the container. It won't appear in `docker compose logs` or similar. Please raise a PR or an issue if you have a suggestion on how to forward the output, even if only so the user can tell when the rebuild is finished.
- There are inevitably a bunch of applications that will never work in a container and systemd related settings that will break things.
- Error noise all over the place I haven't managed to surpress
- Untested on Windows or MacOS hosts. YMMV. It would need a special build for non-emulated M1 support.

## Thanks

Special thanks to Christian Stewart (@paralin) who did most of the hard work getting NixOS running in a container in the first place with a clever two stage build process.


## Advanced usages

If you want to load an arbitrary configuration into the container you can do it via a flake rather than a single `configuration.nix`. To use a flake you can specify a standard system flake with `nixosConfigurations` like you would for your desktop or server. This is controlled by `options.nix` in the root directory:


`options.nix`


```
{
    flakeUrl = "github:MyUser/myflakeflake/main";
    nixosConfiguration = "myConfigurationName";
}

```


`flake.nix`


```nix
{
  description = "Flake for a container";

  inputs = {
    nixpkgs.url = "github:NixOS/nixpkgs/nixos-24.05";
  };

  outputs = { self, nixpkgs, ... }@inputs: 
	{
    nixosConfigurations.myConfigurationName = nixpkgs.lib.nixosSystem {
      system = "x86_64-linux";
      modules = [
        ./module
      ];
    };
  };
}

```

Then mount `options.nix` at the root of the container.


```yaml
services:
    nginx:
        image: trpb/docker-nixos:latest
        volumes:
            - type: tmpfs
              target: /run
            - /sys/fs/cgroup:/sys/fs/cgroup:rw
            # When options.nix is defined it will use the flake described in the file
            - ./options.nix:/options.nix
            - ./path/to/web/root:/web
        ports:
            - 80:80
        cgroup: host
```

You're now runing the system configuration described in the flake in the container!