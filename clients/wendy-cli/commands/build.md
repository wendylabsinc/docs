Builds your WendyOS app. Leverages `wendy.json` for metadata. If `wendy.json` is missing, it prompts you to generate one.

- Uses `swift build` for projects with a `Package.swift`
- Uses `docker build` for Dockerfile

If multiple targets are available (both Swift and Dockerfile), you get a menu to select which one to use.

The build command is mainly used to verify your app can build/compile.