# ansible-learning-template
Template for learning ansible

1. Add an environment variable that points to the folder containing your SSH keys (`id_rsa`, `id_rsa.pub`) for connecting to the server via SSH by running the following command in PowerShell: `[System.Environment]::SetEnvironmentVariable("DEFAULT_SSH_KEY", "<path_to_ssh_keys>", "User")`  
After setting the variable, restart VS Code, making sure to close all other open VS Code windows (including those with other projects) so that the changes take effect.

2. Verify that the environment variable is set correctly by running the following command in PowerShell: `$env:DEFAULT_SSH_KEY`

3. Launch the isolated development environment using VS Code Remote - Containers:
    1. Install the extension [VS Code: Remote - Containers](https://marketplace.visualstudio.com/items?itemName=ms-vscode-remote.remote-containers)
    2. In VS Code, press F1 and select Remote-Containers: Reopen in Container

4. Run the Ansible playbook: `ansible-playbook playbooks/main.yml` or `bash run.sh`