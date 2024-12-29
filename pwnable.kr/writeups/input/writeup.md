level has several specific components:
#include <string.h>
#include <sys/socket.h>
#include <arpa/inet.h>

int main(int argc, char* argv[], char* envp[]){
        printf("Welcome to pwnable.kr\n");
        printf("Let's see if you know how to give input to program\n");
        printf("Just give me correct inputs then you will get the flag :)\n");

        // argv
        if(argc != 100) return 0;
        if(strcmp(argv['A'],"\x00")) return 0;
        if(strcmp(argv['B'],"\x20\x0a\x0d")) return 0;
        printf("Stage 1 clear!\n");

        // stdio
        char buf[4];
        read(0, buf, 4);
        if(memcmp(buf, "\x00\x0a\x00\xff", 4)) return 0;
        read(2, buf, 4);
        if(memcmp(buf, "\x00\x0a\x02\xff", 4)) return 0;
        printf("Stage 2 clear!\n");

        // env
        if(strcmp("\xca\xfe\xba\xbe", getenv("\xde\xad\xbe\xef"))) return 0;
        printf("Stage 3 clear!\n");

        // file
        FILE* fp = fopen("\x0a", "r");
        if(!fp) return 0;
        if( fread(buf, 4, 1, fp)!=1 ) return 0;
        if( memcmp(buf, "\x00\x00\x00\x00", 4) ) return 0;
        fclose(fp);
        printf("Stage 4 clear!\n");

        // network
        int sd, cd;
        struct sockaddr_in saddr, caddr;
        sd = socket(AF_INET, SOCK_STREAM, 0);
        if(sd == -1){
                printf("socket error, tell admin\n");
                return 0;
        }
        saddr.sin_family = AF_INET;
        saddr.sin_addr.s_addr = INADDR_ANY;
        saddr.sin_port = htons( atoi(argv['C']) );
        if(bind(sd, (struct sockaddr*)&saddr, sizeof(saddr)) < 0){
                printf("bind error, use another port\n");
                return 1;
        }
        listen(sd, 1);
        int c = sizeof(struct sockaddr_in);
        cd = accept(sd, (struct sockaddr *)&caddr, (socklen_t*)&c);
        if(cd < 0){
                printf("accept error, tell admin\n");
                return 0;
        }
        if( recv(cd, buf, 4, 0) != 4 ) return 0;
        if(memcmp(buf, "\xde\xad\xbe\xef", 4)) return 0;
        printf("Stage 5 clear!\n");

        // here's your flag
        system("/bin/cat flag");
        return 0;
}

argv:
- Need to provide exactly 99 arguments (99 + program name), argument 65/A should be "\x00" and 66/B should be "\x20\x0a\x0d"

initial exploit looks like this:
#include <stdio.h>
#include <stdlib.h>
#include <unistd.h>
#define INPUT_SIZE 100

int main(int argc, char* argv[]) {
    const char *input_argv[INPUT_SIZE + 1];
    input_argv[0] = "/home/input2/input";
    for (int i = 1; i <= INPUT_SIZE; i++) {
        if (i == 'A') {
            input_argv[i] = "\x00";
        } else if (i == 'B') {
            input_argv[i] = "\x20\x0a\x0d";
        } else if (i == INPUT_SIZE) {
            input_argv[i] = NULL; 
        } else {
            input_argv[i] = "\xAA";
        }
    }
    execve(input_argv[0], (char* const*)input_argv, NULL);
    return 1;
}

pipes/stdout,stdin,stderr:
- **pipe**() creates a pipe, a unidirectional data channel that can be
       used for interprocess communication.  The array _pipefd_ is used to
       return two file descriptors referring to the ends of the pipe.
       _pipefd[0]_ refers to the read end of the pipe.  _pipefd[1]_ refers
       to the write end of the pipe.  Data written to the write end of
       the pipe is buffered by the kernel until it is read from the read
       end of the pipe.
- Using pipe() we can create a pipe for both ends that will connect the main exploit with input and with both write edges we could write specific information that will be read from the reading side of the pipe by input
- Using fork() we can create cases for 2 processes - main process (exploit) and child process (input). In the case of the child process we just need to close all pipe handles (not used directly) and call execve() to execute input, and in the case of the parent process we will close the reading handles for both pipes, use the write handles to write the needed data into the pipes and wait until the child process finishes executing input
- The **dup2**() system call performs the same task as **dup**(), but
       instead of using the lowest-numbered unused file descriptor, it
       uses the file descriptor number specified in _newfd_.  In other
       words, the file descriptor _newfd_ is adjusted so that it now
       refers to the same open file description as _oldfd_.

new script looks like this:
#include <stdio.h>
#include <stdlib.h>
#include <unistd.h>
#include <sys/wait.h>
#include <string.h>
#include <sys/socket.h>
#include <arpa/inet.h>
#define INPUT_SIZE 100

int main(int argc, char* argv[]) {
    int pipe_stdin[2];  // Pipe for stdin
    int pipe_stderr[2]; // Pipe for stderr

    if (pipe(pipe_stdin) == -1 || pipe(pipe_stderr) == -1) {
        return 0;
    }
     int pid = fork();
     if (pid == 0) {  // Child process
        dup2(pipe_stdin[0], 0);  // Replace stdin with pipe read end
        dup2(pipe_stderr[0], 2); // Replace stderr with pipe read end
        close(pipe_stdin[1]);
        close(pipe_stdin[0]);
        close(pipe_stderr[1]);
        close(pipe_stderr[0]);
        const char *input_argv[INPUT_SIZE + 1];
        input_argv[0] = "/home/input2/input";
        for (int i = 1; i <= INPUT_SIZE; i++) {
            if (i == 'A') {
                input_argv[i] = "\x00";
            } else if (i == 'B') {
                input_argv[i] = "\x20\x0a\x0d";
            } else if (i == INPUT_SIZE) {
                input_argv[i] = NULL; 
            } else {
                input_argv[i] = "\xAA";
            }
        }
        execve(input_argv[0], (char* const*)input_argv, NULL);
    }
    else {  // Parent process
        close(pipe_stdin[0]);
        close(pipe_stderr[0]);
        write(pipe_stdin[1], "\x00\x0a\x00\xff", 4);
        write(pipe_stderr[1], "\x00\x0a\x02\xff", 4);
        close(pipe_stdin[1]);
        close(pipe_stderr[1]);
        wait(NULL);
    }
    return 1;
}

environment variables:
- As an additional parameter, main() function can receive envp which is a list of all environment variables in syntax of {"name1=val1", "name2=val2" ...} and so on. getenv() gets a key/name of an environment variable so we can pass a list of environment variables to input including a variable with the required name and value 

updated code:
#include <stdio.h>
#include <stdlib.h>
#include <unistd.h>
#include <sys/wait.h>
#include <string.h>
#include <sys/socket.h>
#include <arpa/inet.h>
#define INPUT_SIZE 100

int main(int argc, char* argv[]) {
    int pipe_stdin[2];  // Pipe for stdin
    int pipe_stderr[2]; // Pipe for stderr

    if (pipe(pipe_stdin) == -1 || pipe(pipe_stderr) == -1) {
        return 0;
    }
     int pid = fork();
     if (pid == 0) {  // Child process
        dup2(pipe_stdin[0], 0);  // Replace stdin with pipe read end
        dup2(pipe_stderr[0], 2); // Replace stderr with pipe read end
        close(pipe_stdin[1]);
        close(pipe_stdin[0]);
        close(pipe_stderr[1]);
        close(pipe_stderr[0]);
        const char *input_argv[INPUT_SIZE + 1];
        input_argv[0] = "/home/input2/input";
        for (int i = 1; i <= INPUT_SIZE; i++) {
            if (i == 'A') {
                input_argv[i] = "\x00";
            } else if (i == 'B') {
                input_argv[i] = "\x20\x0a\x0d";
            } else if (i == INPUT_SIZE) {
                input_argv[i] = NULL; 
            } else {
                input_argv[i] = "\xAA";
            }
        }
        const char *input_envp[2];
        input_envp[0] = "\xde\xad\xbe\xef\x3d\xca\xfe\xba\xbe"; 
        input_envp[1] = NULL; 
        execve(input_argv[0], (char* const*)input_argv, (char* const*)input_envp);
    }
    else {  // Parent process
        close(pipe_stdin[0]);
        close(pipe_stderr[0]);
        write(pipe_stdin[1], "\x00\x0a\x00\xff", 4);
        write(pipe_stderr[1], "\x00\x0a\x02\xff", 4);
        close(pipe_stdin[1]);
        close(pipe_stderr[1]);
        wait(NULL);
    }
    return 1;
}

files:
- File should be named "\x0a" and we should write "\x00" 4 times into the file. Initially some problems occurred with creating the file but when making my own directory inside tmp and running the exploit from inside it fixed the problems

Updated script looks like this:
#include <stdio.h>
#include <stdlib.h>
#include <unistd.h>
#include <sys/wait.h>
#include <string.h>
#include <sys/socket.h>
#include <arpa/inet.h>
#include <errno.h>
#define INPUT_SIZE 100

int main(int argc, char* argv[]) {
    int pipe_stdin[2];  // Pipe for stdin
    int pipe_stderr[2]; // Pipe for stderr

    if (pipe(pipe_stdin) == -1 || pipe(pipe_stderr) == -1) {
        return 0;
    }
     int pid = fork();
     if (pid == 0) {  // Child process
        dup2(pipe_stdin[0], 0);  // Replace stdin with pipe read end
        dup2(pipe_stderr[0], 2); // Replace stderr with pipe read end
        close(pipe_stdin[1]);
        close(pipe_stdin[0]);
        close(pipe_stderr[1]);
        close(pipe_stderr[0]);
        const char *input_argv[INPUT_SIZE + 1];
        input_argv[0] = "/home/input2/input";
        for (int i = 1; i <= INPUT_SIZE; i++) {
            if (i == 'A') {
                input_argv[i] = "\x00";
            } else if (i == 'B') {
                input_argv[i] = "\x20\x0a\x0d";
            } else if (i == INPUT_SIZE) {
                input_argv[i] = NULL; 
            } else {
                input_argv[i] = "\xAA";
            }
        }
        const char *input_envp[2];
        input_envp[0] = "\xde\xad\xbe\xef\x3d\xca\xfe\xba\xbe"; 
        input_envp[1] = NULL; 
        FILE *fp = fopen("\x0a", "wb");
        if (!fp) {
            printf("fopen\n");
            return 0;
        }
        if (fwrite("\x00\x00\x00\x00", 1, 4, fp) < 4){
            printf("fwrite\n");
            printf("Error details: %s\n", strerror(errno));
            return 0;
        }
        fclose(fp);
        execve(input_argv[0], (char* const*)input_argv, (char* const*)input_envp);
    }
    else {  // Parent process
        close(pipe_stdin[0]);
        close(pipe_stderr[0]);
        write(pipe_stdin[1], "\x00\x0a\x00\xff", 4);
        write(pipe_stderr[1], "\x00\x0a\x02\xff", 4);
        close(pipe_stdin[1]);
        close(pipe_stderr[1]);
        wait(NULL);
    }
    return 1;
}

sockets:
- We should provide some random port to bind on, we should connect to that port from a socket and send it the string "\xde\xad\xbe\xef" to that port provided in index 'C'

final code looks like this:
#include <stdio.h>
#include <stdlib.h>
#include <unistd.h>
#include <sys/wait.h>
#include <string.h>
#include <sys/socket.h>
#include <arpa/inet.h>
#include <errno.h>
#define INPUT_SIZE 100

int main(int argc, char* argv[]) {
    int pipe_stdin[2];  // Pipe for stdin
    int pipe_stderr[2]; // Pipe for stderr

    if (pipe(pipe_stdin) == -1 || pipe(pipe_stderr) == -1) {
        return 0;
    }
     int pid = fork();
     if (pid == 0) {  // Child process
        dup2(pipe_stdin[0], 0);  // Replace stdin with pipe read end
        dup2(pipe_stderr[0], 2); // Replace stderr with pipe read end
        close(pipe_stdin[1]);
        close(pipe_stdin[0]);
        close(pipe_stderr[1]);
        close(pipe_stderr[0]);
        const char *input_argv[INPUT_SIZE + 1];
        input_argv[0] = "/home/input2/input";
        for (int i = 1; i <= INPUT_SIZE; i++) {
            if (i == 'A') {
                input_argv[i] = "\x00";
            } else if (i == 'B') {
                input_argv[i] = "\x20\x0a\x0d";
            } else if (i == 'C') {
                input_argv[i] = "5678";
            } else if (i == INPUT_SIZE) {
                input_argv[i] = NULL;
            } else {
                input_argv[i] = "\xAA";
            }
        }
        const char *input_envp[3];
        input_envp[0] = "\xde\xad\xbe\xef\x3d\xca\xfe\xba\xbe";
        input_envp[1] = "PATH=/home/input2:/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin";
        input_envp[2] = NULL;
        FILE *fp = fopen("\x0a", "wb");
        if (!fp) {
            printf("fopen\n");
            return 0;
        }
        if (fwrite("\x00\x00\x00\x00", 1, 4, fp) < 4){
            printf("fwrite\n");
            printf("Error details: %s\n", strerror(errno));
            return 0;
        }
        fclose(fp);
        system("ln -s /home/input2/flag flag");  // flag not found, so create symbolic link so it would be found
        system("chmod 777 /tmp/ss");  // Make the symbolic link (created in current directory, mine is /tmp/ss) accessible for input program
        execve(input_argv[0], (char* const*)input_argv, (char* const*)input_envp);
    }
    else {  // Parent process
        close(pipe_stdin[0]);
        close(pipe_stderr[0]);
        write(pipe_stdin[1], "\x00\x0a\x00\xff", 4);
        write(pipe_stderr[1], "\x00\x0a\x02\xff", 4);
        close(pipe_stdin[1]);
        close(pipe_stderr[1]);
        wait(NULL);
    }
    return 1;
}

Python code that sends the needed payload for part 5, runs on host and sends via used port and server IP address:
import socket  
  
# Server details  
SERVER_IP = "128.61.240.205"  
PORT = 5678  
MESSAGE = b"\xde\xad\xbe\xef"  
  
def main():  
    while True:  
        try:  
            # Create socket  
            with socket.socket(socket.AF_INET, socket.SOCK_STREAM) as s:  
                s.connect((SERVER_IP, PORT))  
                s.sendall(MESSAGE)  
                print("Sent 4 bytes successfully")  
  
        except (ConnectionRefusedError, TimeoutError) as e:  
            continue  # Retry on failure  
  
if __name__ == "__main__":  
    main()
    
An important point of the challenge was that although I passed all 5 prints, system(/bin/cat flag) did not do anything / the command failed. To understand that I used strace -f to see output for both parent and child processes. Some main parts were:
- In strace, you can see that when open() is performed on flag (relative path while /bin/cat is an absolute path), flag is not recognized as a file because it is a relative path and it is not relative to the current working directory
- To know that I saw the most recent call to getcwd() - current working directory, which came out as "/tmp/ss" (the directory I executed the exploit from)
- Another thought I had is to add the absolute parent directory, "/home/input2" to a PATH variable that is passed to input, but it makes no sense as only the programs are searched through the PATH variable, not the arguments themselves, so using environment variables would not work
- Because PATH environment variable would not help in this case and because I cannot change the current working directory to be relative to path, I understood that I need to create something in the current working directory which I also have complete access to ("/tmp/ss"), that will point to /home/input2/flag that should be also named flag and that I could create with no special permissions - SYMBOLIC LINKS. I do not need special permissions to the destination to create a symbolic link (unlike a hard link), it will be the "flag" file that exists in the current working directory and will point to the actual flag file I want to read and that input program has access to. For that i added 2 commands at the end before execve() - creating the symbolic link, and changing permissions to /tmp/ss so everyone will be able to completely access the directory

Relevant strace output:
![[asdasdasdasds.png]]

input2@pwnable:/tmp/ss$ ./s
Welcome to pwnable.kr
Let's see if you know how to give input to program
Just give me correct inputs then you will get the flag :)
Stage 1 clear!
Stage 2 clear!
Stage 3 clear!
Stage 4 clear!
Stage 5 clear!
Mommy! I learned how to pass various input in Linux :)