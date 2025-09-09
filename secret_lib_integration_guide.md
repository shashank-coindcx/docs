# Secret Library Integration Guide
1. Update the code
2. Add scripts
3. Update Makefile
4. Update Dockerfile
5. Update values.yaml
6. Get Service Account being set up for the migration and update the helm if required

### Update the code
- Add the code in the init function of your application to initialize the secret library.
- Keep the code such that it has backward compatibility.

### Add scripts
- Create a directory named `scripts` in the root of your project if it doesn't exist
- Add the following script as `scripts/migrate_up.sh`:
    ```bash
    #!/bin/bash
    eval $(./secret)
    migrate -path migrations/postgres -database $POSTGRES_MASTER_URL -verbose up
    ```
- Add the following script as `scripts/migrate_down.sh`:
    ```bash
    #!/bin/bash
    eval $(./secret)
    migrate -path migrations/postgres -database $POSTGRES_MASTER_URL -verbose down
    ```

### Update Makefile
- Update the `migrate_up` target in your Makefile to use the new script:
    ```Makefile
    migrate_up:
        $(PWD)/scripts/migrate_up.sh
    ```

- Update the `migrate_down` target in your Makefile to use the new script:
    ```Makefile
    migrate_down:
        $(PWD)/scripts/migrate_down.sh
    ```

### Update Dockerfile
- Add the following lines to your Dockerfile in build stage after `RUN sh dependencyfile.sh` to install the secret library:
    ```Dockerfile
    RUN GOOS=linux GOARCH=${TARGETARCH} CGO_ENABLED=1 CGO_LDFLAGS="-lsasl2" go build -tags musl -ldflags "-w -s" -o secret \
        $GOPATH/pkg/mod/github.com/coindcx-app/common-modules-golang/secret@*/cmd/secret
    ```
  
- Add the following lines to copy the secret binary to the final image:
    ```Dockerfile
    COPY --from=build --chown=dcx:dcx --chmod=750 /go/src/secret ./secret
    COPY --chown=dcx:dcx --chmod=750 ./scripts ./scripts
    ```
  
### Update values.yaml
- Remove all the secrets from `values.yaml`
- Add `APP_SECRET_NAMES: '["key1","key2"...]'` to `env` section in `values.yaml` where `key1`, "`key2` are the names of the secrets you want to fetch from the secret management system.
  
