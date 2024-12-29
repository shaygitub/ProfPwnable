#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <fcntl.h>

int main(int argc, char* argv[]){
        char fname[128];
        unsigned long long otp[2];

        if(argc!=2){
                printf("usage : ./otp [passcode]\n");
                return 0;
        }

        int fd = open("/dev/urandom", O_RDONLY);
        if(fd==-1) exit(-1);

        if(read(fd, otp, 16)!=16) exit(-1);
        close(fd);

        sprintf(fname, "/tmp/%llu", otp[0]);
        FILE* fp = fopen(fname, "w");
        if(fp==NULL){ exit(-1); }
        fwrite(&otp[1], 8, 1, fp);
        fclose(fp);

        printf("OTP generated.\n");

        unsigned long long passcode=0;
        FILE* fp2 = fopen(fname, "r");
        if(fp2==NULL){ exit(-1); }
        fread(&passcode, 8, 1, fp2);
        fclose(fp2);

        if(strtoul(argv[1], 0, 16) == passcode){
                printf("Congratz!\n");
                system("/bin/cat flag");
        }
        else{
                printf("OTP mismatch\n");
        }

        unlink(fname);
        return 0;
}

Summarized code:
- Reads 16 bytes from /dev/urandom that generates random data
- Uses first 8 bytes as name for temp file in /tmp/{val}
- Writes second 8 bytes into file, closes file
- Opens file again for reading , reads 8 bytes
- Checks if argv[1] == value read from file

solution:
- Create limit for writing into file (0 bytes, cannot write anything)
- Use SIG_IGN to ignore signal to stop otp from crashing
- Send 0 as passcode is also zero

solution:
#include <stdio.h>

#include <stdlib.h>

#include <unistd.h>

#include <signal.h>

#include <sys/resource.h>

#include <fcntl.h>

#include <string.h>

#include <sys/wait.h>

#include <errno.h>

  
  

int main() {

    // Ignore SIGXFSZ signal

    struct sigaction sa;

    sa.sa_handler = SIG_IGN; // Ignore the signal

    sa.sa_flags = 0;         // No special flags

    sigemptyset(&sa.sa_mask);

    if (sigaction(SIGXFSZ, &sa, NULL) != 0) {

        perror("sigaction failed");

        exit(EXIT_FAILURE);

    }

  

    pid_t pid = fork();

  

    if (pid < 0) {

        perror("fork failed");

        exit(EXIT_FAILURE);

    }

  

    if (pid == 0) {

        // Child process

        printf("Child process (PID: %d)\n", getpid());

        const char* input_argv[3];

        input_argv[0] = "/home/otp/otp";

        input_argv[1] = "0";

        input_argv[2] = NULL;

  

        // Set RLIMIT_FSIZE to 0 to prevent any file growth

        struct rlimit rl;

        rl.rlim_cur = 0;  // Soft limit

        rl.rlim_max = 0;  // Hard limit

        if (setrlimit(RLIMIT_FSIZE, &rl) != 0) {

            perror("setrlimit failed");

            exit(EXIT_FAILURE);

        }

  

        execve(input_argv[0], (char* const*)input_argv, NULL);

  

    } else {

        // Parent process

        printf("Parent process (PID: %d)\n", getpid());

        int status;

        if (waitpid(pid, &status, 0) == -1) {

            perror("waitpid failed");

            exit(EXIT_FAILURE);

        }

        if (WIFEXITED(status)) {

            printf("Child exited with status %d\n", WEXITSTATUS(status));

        } else if (WIFSIGNALED(status)) {

            printf("Child terminated by signal %d\n", WTERMSIG(status));

        } else {

            printf("Child exited with unknown status\n");

        }

    }

  

    return 0;

}
