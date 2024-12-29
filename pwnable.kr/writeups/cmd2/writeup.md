Code is as follows:
#include <stdio.h>
#include <string.h>

int filter(char* cmd){
        int r=0;
        r += strstr(cmd, "=")!=0;
        r += strstr(cmd, "PATH")!=0;
        r += strstr(cmd, "export")!=0;
        r += strstr(cmd, "/")!=0;
        r += strstr(cmd, "`")!=0;
        r += strstr(cmd, "flag")!=0;
        return r;
}

extern char** environ;
void delete_env(){
        char** p;
        for(p=environ; *p; p++) memset(*p, 0, strlen(*p));
}

int main(int argc, char* argv[], char** envp){
        delete_env();
        putenv("PATH=/no_command_execution_until_you_become_a_hacker");
        if(filter(argv[1])) return 0;
        printf("%s\n", argv[1]);
        system( argv[1] );
        return 0;
}

Cannot use '/', '=', "flag", "export", "PATH" or "`"
It deletes every environment variable including PATH so cannot use any program, decided to try to work with bash scripts that use default symbols and the read command.

 ./cmd2 "for file in *; do   if [ -f \$file ]; then     echo Contents of \$file:;     while read -r line; do       echo \$line;     done < \$file;     echo;   fi; done"

Actual used script looks like:
for file in *; do
  if [ -f \$file ]; then
    echo Contents of \$file:
    while read -r line; do
      echo \$line
    done < \$file
    echo
  fi
done

Something i had to figure out is that $ will not work as expected when not prefixed with a backslash as the program expects it will be a special shell variable and so the value is redacted and "$file" will be reduced to " ".

![[asdasdsadasdasdasd 1.png]]