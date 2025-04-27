# Flyway operator

This is from https://github.com/davidkarlsen/flyway-operator/blob/main/INSTALLING.md

> helm repo add flyway-operator https://davidkarlsen.github.io/flyway-operator
> helm template flyway-operator/flyway-operator > .\flyway-operator.yaml

Manual stuff done

* Replace `release-name` with `huemie`
* Remove `app.kubernetes.io/managed-by`
* Remove `helm.sh/chart`

Then I run the CRD updates from the GitHub repositories

# Mariadb

Sets up database as well as initiat permissions used by the local environment
