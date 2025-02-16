# Fault-tolerant DNS configuration for subnets in a VPC

This solution allows you to configure a DNS service in high-availability mode when using multiple availability zones.

## Prerequisites

* You have an existing Yandex Cloud folder.
* You have installed [**yc**](https://yandex.cloud/docs/cli/quickstart).

## About the solution

This solution ensures high availability by redirecting DNS requests to a standby DNS service in a different zone if the active DNS service in the local availability zone fails.

While following this example, you will be creating the following artifacts in the Yandex Cloud infrastructure:
1. **VPC**: Shared object across all availability zones (specify network attributes in input parameters).
1. **IPv4 subnet or subnets**: One for each availability zone (specify subnet attributes in input parameters). The target service is assumed to be highly available and deployed across at least two availability zones.

When you create a default subnet, the system automatically adds the second IP address from the subnet's IPv4 prefix to its DNS server list. For more information, see _Subnets_ in [Cloud networks and subnets](https://yandex.cloud/docs/vpc/concepts/network).

When creating an IPv4 subnet, the `domain_name_servers` attribute is automatically configured to include a list of IPv4 addresses for DNS servers located in different availability zones. For example, let’s assume you configured your subnets as follows:

```json
[
    { name = "sub1", zone = "ru-central1-a", prefix = "10.1.1.128/25" },
    { name = "sub2", zone = "ru-central1-b", prefix = "10.2.2.0/24" },
    { name = "sub3", zone = "ru-central1-c", prefix = "10.3.3.64/28" },
]
```

In this case, the list of DNS servers for each of the subnets above will look like this:

```yaml
sub1: [10.1.1.130, 10.2.2.2, 10.3.3.66]
sub2: [10.2.2.2, 10.1.1.130, 10.3.3.66]
sub3: [10.3.3.66, 10.1.1.130, 10.2.2.2]
```

## Usage

### Input parameters for [`variables.tf`](./variables.tf)
* `net_name`: VPC name in Yandex Cloud. The network will automatically be created in all availability zones.
* `subnet_list`: List of IPv4 subnets that will be created in the network specified in `net_name`. Each subnet is defined by the following attributes:
    * `name`: Subnet name.
    * `zone`: Availability zone name.
    * `prefix`: IPv4 prefix of the subnet being created.

### Deployment
1. In [`env-yc-prod.sh`](./env-yc-prod.sh), check that the YC runtime environment is properly configured. Rename the `prod` profile as required.
1. Activate the environment:

    ```bash
    source env-yc-prod.sh
    ```

1. Initialize Terraform:

    ```bash
    terraform init
    ```

1. Create subnets in the specified network with a fault-tolerant DNS service:

    ```bash
    terraform apply
    ```

### Testing

1. Save the external address of the created VM to an environment variable:

    ```bash
    export vm_ext_ip=`terraform output vm_ext_ip_address | sed 's/"//g'`
    ```

1. Connect to the VM:

    ```bash
    ssh -oStrictHostKeyChecking=no ubuntu@$vm_ext_ip
    ```

1. Test the DNS service:

    ```bash
    dig www.yandex.ru
    ```

1. With `iptables`, disable all three DNS servers:

    ```bash
    sudo iptables -A OUTPUT -d 10.1.1.130 -j DROP
    sudo iptables -A OUTPUT -d 10.2.2.2 -j DROP
    sudo iptables -A OUTPUT -d 10.3.3.66 -j DROP
    sudo iptables -L -v -n
    ```

1. Make sure DNS requests now get no response. This is what we expect since the DNS service is disabled:

    ```bash
    dig www.yandex.ru
    ```

1.  Allow traffic to the second DNS server in the list by deleting the deny rule we created in _Step 4_:

    ```bash
    sudo iptables -D OUTPUT -d 10.2.2.2 -j DROP
    ```

1. Make sure the DNS service is up and running again:

    ```bash
    dig www.yandex.ru
    ```

### Deleting the created resources

```bash
unset vm_ext_ip
terraform destroy
```
