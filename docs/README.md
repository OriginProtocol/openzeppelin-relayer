# Generate Documentation

## Deployment note

- The Railway production service is configured with GitHub auto-deploy on the `production` branch.
- A push to `production` automatically triggers Docker build and deployment.

- To generate rust documentation locally, run the following command

  - In separate terminal from root of the repo run:

    ```sh
    cargo make rust-docs
    ```

- To update openapi documentation, run:

  ```sh
  cargo generate_openapi
  ```
