## Intercepting-Docker-CLI UNIX socket requests using Burp

Docker is now a household name in container technology, and for good reason. The Docker CLI makes managing containers and everything related to them incredibly simple and efficient, making it a go-to tool in my workflow.

Docker CLI uses REST API to connect to Docker Daemon (dockerd), which by default listens for requests on the UNIX socket, ideally located at `/var/run/docker.sock`, to only allow local connections by the root user. We do have another option to expose a TCP port and proxy curl command through it.

---

### This is a header

#### Some T-SQL Code

```tsql
SELECT This, [Is], A, Code, Block -- Using SSMS style syntax highlighting
    , REVERSE('abc')
FROM dbo.SomeTable s
    CROSS JOIN dbo.OtherTable o;
```

#### Some PowerShell Code

```powershell
Write-Host "This is a powershell Code block";

# There are many other languages you can use, but the style has to be loaded first

ForEach ($thing in $things) {
    Write-Output "It highlights it using the GitHub style"
}
```
