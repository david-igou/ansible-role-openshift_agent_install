# david-igou.openshift_agent_install

Downloads `openshift-install` and generates agent-based installer boot artifacts (PXE files or ISO image) for OpenShift Container Platform.

## Requirements

- Ansible >= 2.15
- `oc` client matching the target OpenShift version must be available on `PATH` (the role does not install it)
- Internet access to download the `openshift-install` tarball and pull release metadata

## Role Variables

### Required

| Variable | Description |
|---|---|
| `openshift_agent_install_version` | OpenShift version to install (e.g. `"4.20.14"`) |
| `openshift_agent_install_config` | Dictionary representing `install-config.yaml` content |
| `openshift_agent_install_agent_config` | Dictionary representing `agent-config.yaml` content |

### Optional

| Variable | Default | Description |
|---|---|---|
| `openshift_agent_install_arch` | `"x86_64"` | Architecture for the `openshift-install` binary |
| `openshift_agent_install_mirror_url` | `"https://mirror.openshift.com/pub/openshift-v4"` | Base mirror URL |
| `openshift_agent_install_tarball_url` | *(constructed from version, arch, mirror)* | Full URL to the tarball. Override for custom mirrors or OKD |
| `openshift_agent_install_work_dir` | `"{{ ansible_env.HOME }}/openshift-agent-install"` | Working directory for all artifacts |
| `openshift_agent_install_output_type` | `"pxe-files"` | Output type: `"pxe-files"` for PXE/iPXE boot artifacts, `"image"` for a bootable agent ISO |
| `openshift_agent_install_verify_checksum` | `true` | Verify SHA256 checksum of the downloaded tarball. Disable for sources without `sha256sum.txt` |
| `openshift_agent_install_checksum_url` | *(constructed from version, arch, mirror)* | URL to `sha256sum.txt` for checksum verification. Override for OKD or custom mirrors |
| `openshift_agent_install_validate_oc` | `true` | Validate that `oc` is on PATH before running `openshift-install` |
| `openshift_agent_install_additional_manifests` | `[]` | List of extra manifests to place in `openshift/` directory |

### Additional Manifests Format

```yaml
openshift_agent_install_additional_manifests:
  - file_name: my-machineconfig.yml
    contents: |
      apiVersion: machineconfiguration.openshift.io/v1
      kind: MachineConfig
      metadata:
        name: 99-custom
      spec: ...
```

## Tags

| Tag | Description |
|---|---|
| `download` | Download and extract the `openshift-install` binary |
| `manifests` | Render `install-config.yaml`, `agent-config.yaml`, and additional manifests |
| `install` | Run `openshift-install agent create pxe-files` or `agent create image` depending on `output_type` |
| `cleanup` | Remove the working directory (never runs by default) |

## Example Playbook

### OCP (Red Hat OpenShift)

```yaml
- hosts: localhost
  connection: local
  gather_facts: true
  roles:
    - role: david-igou.openshift_agent_install
      vars:
        openshift_agent_install_version: "4.20.14"
        openshift_agent_install_config:
          apiVersion: v1
          baseDomain: example.com
          metadata:
            name: my-cluster
          compute:
            - architecture: amd64
              hyperthreading: Enabled
              name: worker
              replicas: 0
          controlPlane:
            architecture: amd64
            hyperthreading: Enabled
            name: master
            replicas: 1
          networking:
            clusterNetwork:
              - cidr: 10.128.0.0/14
                hostPrefix: 23
            machineNetwork:
              - cidr: 192.168.0.0/16
            networkType: OVNKubernetes
            serviceNetwork:
              - 172.30.0.0/16
          platform:
            none: {}
          pullSecret: '{{ pull_secret }}'
          sshKey: '{{ ssh_pub_key }}'
        openshift_agent_install_agent_config:
          apiVersion: v1beta1
          kind: AgentConfig
          metadata:
            name: my-cluster
          rendezvousIP: 192.168.111.80
          bootArtifactsBaseURL: http://pxe-server.example.com/my-cluster/
```

### OKD

Override `openshift_agent_install_tarball_url` and `openshift_agent_install_checksum_url` to pull from the OKD GitHub releases:

```yaml
- hosts: localhost
  connection: local
  gather_facts: true
  vars:
    okd_version: "4.21.0-okd-scos.6"
    okd_release_url: "https://github.com/okd-project/okd/releases/download/{{ okd_version }}"
  roles:
    - role: david-igou.openshift_agent_install
      vars:
        openshift_agent_install_version: "{{ okd_version }}"
        openshift_agent_install_tarball_url: "{{ okd_release_url }}/openshift-install-linux-{{ okd_version }}.tar.gz"
        openshift_agent_install_checksum_url: "{{ okd_release_url }}/sha256sum.txt"
        openshift_agent_install_config:
          apiVersion: v1
          baseDomain: example.com
          metadata:
            name: my-okd-cluster
          # ... rest of config
        openshift_agent_install_agent_config:
          apiVersion: v1beta1
          kind: AgentConfig
          metadata:
            name: my-okd-cluster
          rendezvousIP: 192.168.111.80
```

### Generating an ISO Instead of PXE Files

Set `openshift_agent_install_output_type` to `"image"` to produce a bootable agent ISO:

```yaml
- hosts: localhost
  connection: local
  gather_facts: true
  roles:
    - role: david-igou.openshift_agent_install
      vars:
        openshift_agent_install_version: "4.20.14"
        openshift_agent_install_output_type: "image"
        openshift_agent_install_config:
          apiVersion: v1
          baseDomain: example.com
          metadata:
            name: my-cluster
          compute:
            - architecture: amd64
              hyperthreading: Enabled
              name: worker
              replicas: 0
          controlPlane:
            architecture: amd64
            hyperthreading: Enabled
            name: master
            replicas: 1
          networking:
            clusterNetwork:
              - cidr: 10.128.0.0/14
                hostPrefix: 23
            machineNetwork:
              - cidr: 192.168.0.0/16
            networkType: OVNKubernetes
            serviceNetwork:
              - 172.30.0.0/16
          platform:
            none: {}
          pullSecret: '{{ pull_secret }}'
          sshKey: '{{ ssh_pub_key }}'
        openshift_agent_install_agent_config:
          apiVersion: v1beta1
          kind: AgentConfig
          metadata:
            name: my-cluster
          rendezvousIP: 192.168.111.80
```

The ISO will be written to `<work_dir>/cluster-manifests/agent.<arch>.iso`.

## Running Specific Phases

```bash
# Only download the binary
ansible-playbook site.yml --tags download

# Only render manifests
ansible-playbook site.yml --tags manifests

# Clean up the working directory
ansible-playbook site.yml --tags cleanup
```

## Testing

Tests use [Molecule](https://molecule.readthedocs.io/) with Podman and OKD binaries:

```bash
cd ansible-role-openshift_agent_install
molecule test
```

## License

MIT

## Author

David Igou
