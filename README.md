# OpenShift AWS Install The Hard Way

To run straight from the documentation

```bash
# https://gitlab.com/nul.one/rundoc
pip3 install rundoc --user
rundoc run ocp-3.11.md --single-session bash
```

Lucid chart import (once installed)

```
lucid_aws_export.py
cat aws.json  | jq . | cat > aws2.json
```

## Versions
OpenShift 3.11 [here](ocp-3.11.md)

![architecture](images/topology.png)

![architecture](images/aws-ocp.png)

