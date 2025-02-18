## 1. Overall Code Organization

- **Directory Structure**  
  We will reorganize the repository to separate the CLI commands, internal logic, and package definitions. For example:

  ```
  microcks-cli/
  ├── cmd/
  │   ├── root.go         // Cobra root command definition
  │   ├── test.go         // Existing test command (refactored)
  │   ├── import.go       // Existing import command (refactored)
  │   ├── start.go        // New command: start (to launch a Microcks instance)
  │   ├── stop.go         // New command: stop (to shutdown a Microcks instance)
  │   └── ...             // Additional commands can be added here (e.g. import-from-url, create-job, list-jobs)
  ├── internal/
  │   └── mksctl/         // Internal logic for managing instances and CLI-specific utilities
  ├── pkg/
  │   └── ...             // Shared libraries (e.g., HTTP client wrappers, configuration loaders)
  ├── Makefile            // Build and packaging scripts
  ├── go.mod
  └── go.sum
  ```

---

## 2. Root Command with Cobra

- **File: cmd/root.go**  
  Create or refactor the root command to initialize Cobra and define global flags (e.g., configuration file path, logging level). For example:

  ```go
  package cmd

  import (
      "fmt"
      "os"

      "github.com/spf13/cobra"
      "github.com/spf13/viper"
  )

  var cfgFile string

  // rootCmd represents the base command when called without any subcommands.
  var rootCmd = &cobra.Command{
      Use:   "mksctl",
      Short: "Microcks CLI utility for mocking and testing APIs",
      Long: `Microcks CLI (mksctl) provides a unified interface for:
  - Running tests and importing API artifacts.
  - Managing Microcks instances (start/stop).
  - Extending functionality with new commands.
  `,
      Run: func(cmd *cobra.Command, args []string) {
          // If no subcommand is provided, show help.
          cmd.Help()
      },
  }

  // Execute adds all child commands to the root command and sets flags appropriately.
  func Execute() {
      if err := rootCmd.Execute(); err != nil {
          fmt.Println(err)
          os.Exit(1)
      }
  }

  func init() {
      cobra.OnInitialize(initConfig)
      // Global flag for configuration file
      rootCmd.PersistentFlags().StringVar(&cfgFile, "config", "", "config file (default is $HOME/.mksctl.yaml)")
  }

  func initConfig() {
      if cfgFile != "" {
          viper.SetConfigFile(cfgFile)
      } else {
          home, err := os.UserHomeDir()
          if err != nil {
              fmt.Println("Unable to detect home directory:", err)
              os.Exit(1)
          }
          viper.AddConfigPath(home)
          viper.SetConfigName(".mksctl")
      }
      if err := viper.ReadInConfig(); err == nil {
          fmt.Println("Using config file:", viper.ConfigFileUsed())
      }
  }
  ```

- **Global Behavior:**  
  – The root command prints help when no subcommand is provided.  
  – Viper is used to load configuration.

---

## 3. Refactoring Existing Commands

### 3.1 Test and Import Commands

- **File: cmd/test.go**  
  Refactor the existing test command to use Cobra’s flag parsing. For example:

  ```go
  package cmd

  import (
      "fmt"
      "os"
      "time"

      "github.com/spf13/cobra"
  )

  var testCmd = &cobra.Command{
      Use:   "test [APIName:Version] [endpoint]",
      Short: "Execute tests against an API",
      Long:  "Execute a test against the provided API definition using the specified endpoint. Additional flags control authentication, headers, and wait duration.",
      Args:  cobra.ExactArgs(2),
      Run: func(cmd *cobra.Command, args []string) {
          apiSpec := args[0]
          endpoint := args[1]
          microcksURL, _ := cmd.Flags().GetString("microcksURL")
          waitFor, _ := cmd.Flags().GetDuration("waitFor")
          // Here, call the internal test runner logic (in internal/mksctl/test.go)
          err := runTest(apiSpec, endpoint, microcksURL, waitFor)
          if err != nil {
              fmt.Println("Test execution failed:", err)
              os.Exit(1)
          }
          fmt.Println("Test executed successfully")
      },
  }

  func init() {
      rootCmd.AddCommand(testCmd)
      testCmd.Flags().String("microcksURL", "", "Microcks API URL")
      testCmd.Flags().Duration("waitFor", 5*time.Second, "Time to wait for test completion")
      // Mark microcksURL as required if needed:
      // _ = testCmd.MarkFlagRequired("microcksURL")
  }
  ```

- **File: cmd/import.go**  
  Refactor the import command similarly, ensuring consistency with test command flags and error messages.

### 3.2 New Commands: Start and Stop

- **File: cmd/start.go**  
  Implement a command to launch a new Microcks instance locally. This command should call internal logic that starts a container (or launches a local process).

  ```go
  package cmd

  import (
      "fmt"
      "os"

      "github.com/spf13/cobra"
      "microcks-cli/internal/mksctl"
  )

  var startCmd = &cobra.Command{
      Use:   "start",
      Short: "Start a new Microcks instance",
      Long:  "Starts a new instance of Microcks locally (using Docker or a pre-defined binary) for development and testing purposes.",
      Run: func(cmd *cobra.Command, args []string) {
          err := mksctl.StartInstance()
          if err != nil {
              fmt.Println("Error starting Microcks instance:", err)
              os.Exit(1)
          }
          fmt.Println("Microcks instance started successfully")
      },
  }

  func init() {
      rootCmd.AddCommand(startCmd)
      // Additional flags can be added, e.g., for port or configuration overrides.
  }
  ```

- **File: cmd/stop.go**  
  Similarly, implement a stop command that terminates the running instance.

  ```go
  package cmd

  import (
      "fmt"
      "os"

      "github.com/spf13/cobra"
      "microcks-cli/internal/mksctl"
  )

  var stopCmd = &cobra.Command{
      Use:   "stop",
      Short: "Stop the running Microcks instance",
      Long:  "Stops the currently running Microcks instance, cleaning up all related resources.",
      Run: func(cmd *cobra.Command, args []string) {
          err := mksctl.StopInstance()
          if err != nil {
              fmt.Println("Error stopping Microcks instance:", err)
              os.Exit(1)
          }
          fmt.Println("Microcks instance stopped successfully")
      },
  }

  func init() {
      rootCmd.AddCommand(stopCmd)
  }
  ```

### 3.3 Additional Commands for Future Extensions

Create stubs for commands such as:
- **import-from-url** – to import API definitions from a given URL.
- **import-directory** – to import all API files from a directory.
- **create-job** and **list-jobs** – for job management.

For each, define the command in a separate file under `cmd/` with proper flag definitions and a call to the corresponding logic in `internal/mksctl`.

---

## 4. Internal Logic (Package: internal/mksctl)

Implement the actual operations for instance management and command execution in a separate package. For example:

- **File: internal/mksctl/start.go**

  ```go
  package mksctl

  import (
      "errors"
      "os/exec"
  )

  // StartInstance launches a new Microcks instance.
  func StartInstance() error {
      // Example: Use Docker to run the Microcks container.
      cmd := exec.Command("docker", "run", "-d", "-p", "8585:8585", "microcks/microcks:latest")
      err := cmd.Run()
      if err != nil {
          return errors.New("failed to start Docker container: " + err.Error())
      }
      return nil
  }
  ```

- **File: internal/mksctl/stop.go**

  ```go
  package mksctl

  import (
      "errors"
      "os/exec"
  )

  // StopInstance stops the running Microcks instance.
  func StopInstance() error {
      // Example: Stop the container by its name or ID.
      cmd := exec.Command("docker", "stop", "microcks_container")
      err := cmd.Run()
      if err != nil {
          return errors.New("failed to stop Microcks instance: " + err.Error())
      }
      return nil
  }
  ```

- **File: internal/mksctl/test.go**  
  Implement a function (e.g., `runTest(apiSpec, endpoint string, microcksURL string, waitFor time.Duration) error`) that calls the Microcks API to launch a test. This function should:
  - Validate inputs.
  - Make HTTP requests to the Microcks API.
  - Wait for the specified duration and return success or error.

---

## 5. Packaging & Distribution

- **Makefile:**  
  Create targets in the Makefile to:
  - Build the CLI binary.
  - Run tests.
  - Package the binary for distribution (e.g., creating tarballs for Linux/Mac/Windows).
  - Optionally, automate creation of Homebrew formula or APT package scripts.

  Example snippet:

  ```makefile
  .PHONY: build test package

  build:
      go build -o mksctl main.go

  test:
      go test ./...

  package: build
      tar -czvf mksctl-$(shell git describe --tags).tar.gz mksctl
  ```

- **CI/CD Integration:**  
  Ensure the repository includes configuration files (e.g., GitHub Actions workflows) that run tests, build the binary, and create release artifacts.

---

## 6. Testing Strategy

- **Unit Tests:**  
  Write unit tests for internal functions (e.g., start/stop instance functions, test runner logic) in the `internal/mksctl` package.

- **Command Execution Tests:**  
  Utilize helper functions (similar to Cobra’s `ExecuteCommand`) to simulate CLI command execution and capture output.  
  For example, create a `cmd/root_test.go` that executes each command with specific arguments and checks that the output or error codes match expectations.

  ```go
  package cmd

  import (
      "bytes"
      "testing"
  )

  // ExecuteCommand is a helper that sets up command execution.
  func ExecuteCommand(root *cobra.Command, args ...string) (output string, err error) {
      buf := new(bytes.Buffer)
      root.SetOut(buf)
      root.SetErr(buf)
      root.SetArgs(args)
      _, err = root.ExecuteC()
      return buf.String(), err
  }

  func TestStartCommand(t *testing.T) {
      output, err := ExecuteCommand(rootCmd, "start")
      if err != nil {
          t.Fatalf("Expected no error, got %v", err)
      }
      if output == "" {
          t.Fatal("Expected output message, got empty string")
      }
  }
  ```

---

## 7. Flag Handling & Error Management

- **Uniform Flag Parsing:**  
  Each command should use Cobra’s flag methods (e.g., `StringVarP`, `Duration`, etc.) for consistent behavior. Mark required flags where necessary using `MarkFlagRequired`.

- **Error Handling:**  
  On any error (e.g., missing flag values, failure to start/stop instances, HTTP request failures), print a clear error message and exit with a non-zero status code.

- **Help & Usage:**  
  Rely on Cobra’s automatic help generation to provide usage examples and command details.
