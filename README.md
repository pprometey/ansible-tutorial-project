# ansible-tutorial-project

Repository with final artifacts and full commit history documenting the creation of the Ansible basic tutorial course.

The full step-by-step course content can be found at:

- [Ansible Basics in 13 Steps](https://chernyavsky.is-a.dev/devops/guides/iat/Ansible-Basics-in-13-Steps) (English version)  
- [Основы Ansible за 13 шагов](https://chernyavsky.is-a.dev/devops/guides/iat/%D0%9E%D1%81%D0%BD%D0%BE%D0%B2%D1%8B-Ansible-%D0%B7%D0%B0-13-%D1%88%D0%B0%D0%B3%D0%BE%D0%B2) (Russian version)  

To get the most out of the material, it is highly recommended to manually repeat all the steps of the guide by creating the required artifacts yourself. This will help you gain a deeper understanding of the processes and better retain the key concepts.

## How to Use This Environment to Follow the Course

1. **Install Docker Desktop:**

   For Windows, Mac, or Linux — follow the official instructions:  
   <https://docs.docker.com/get-docker/>

2. **Clone the course repository:**

    ```bash
    git clone https://github.com/pprometey/ansible-tutorial-lab.git
    ```

    Change into the cloned repository directory, which will be the root directory of the project:

    ```bash
    cd ansible-tutorial-lab
    ```

3. **Set up the environment variable for SSH keys:**

    - **Windows (PowerShell):**

      ```powershell
      [System.Environment]::SetEnvironmentVariable("DEFAULT_SSH_KEY", "<path_to_ssh_keys_folder>", "User")
      ```

      Then restart all VS Code windows.

    - **Mac/Linux (bash):**  
      Add the following line to your `~/.bashrc` or `~/.zshrc`:

      ```bash
      export DEFAULT_SSH_KEY="<path_to_ssh_keys_folder>"
      ```

      Then run:

      ```bash
      source ~/.bashrc
      ```

      or restart your terminal.

    The specified folder must contain two files named `id_rsa` and `id_rsa.pub` — the private and public SSH keys that Ansible will use to connect to the target nodes.

4. **Verify that the variable is set:**

    - Windows (PowerShell):

      ```powershell
      $env:DEFAULT_SSH_KEY
      ```

    - Mac/Linux (bash):

      ```bash
      echo $DEFAULT_SSH_KEY
      ```

5. **Launch the isolated environment in VS Code with the Remote - Containers extension:**

    - Open VS Code by running `code .`
    - [Install the **Remote - Containers** extension](https://marketplace.visualstudio.com/items?itemName=ms-vscode-remote.remote-containers) if you haven't already.
    - Press `F1` and select **Remote-Containers: Reopen in Container**.  
      This will start containers for the Ansible master and the target nodes.
