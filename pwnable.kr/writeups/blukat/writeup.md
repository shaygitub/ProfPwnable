When using a pipe in linux, both sides will be executed somewhat at the same time and the output (stdout) from the first program will be directed/piped to the input (stdin) of the next program.

In this challenge the password is stored in the file password and the password needs to be passed correctly to blukat program to decrypt the actual flag for the challenge
```c
#include <stdio.h>
#include <string.h>
#include <stdlib.h>
#include <fcntl.h>
char flag[100];
char password[100];
char* key = "3\rG[S/%\x1c\x1d#0?\rIS\x0f\x1c\x1d\x18;,4\x1b\x00\x1bp;5\x0b\x1b\x08\x45+";
void calc_flag(char* s){
	int i;
	for(i=0; i<strlen(s); i++){
		flag[i] = s[i] ^ key[i];
	}
	printf("%s\n", flag);
}
int main(){
	FILE* fp = fopen("/home/blukat/password", "r");
	fgets(password, 100, fp);
	char buf[100];
	printf("guess the password!\n");
	fgets(buf, 128, stdin);
	if(!strcmp(password, buf)){
		printf("congrats! here is your flag: ");
		calc_flag(password);
	}
	else{
		printf("wrong guess!\n");
		exit(0);
	}
	return 0;
}
```

When checking the list of files/directories and permissions in root directory of /home/blukat we could see some interesting things:
blukat@pwnable:~$ ls -alr
total 36
drwxr-xr-x   2 root root       4096 Aug 16  2018 .pwntools-cache
-rw-r-----   1 root blukat_pwn   33 Jan  6  2017 password
dr-xr-xr-x   2 root root       4096 Aug 16  2018 .irssi
-rw-r--r--   1 root root        645 Aug  8  2018 blukat.c
-r-xr-sr-x   1 root blukat_pwn 9144 Aug  8  2018 blukat
drwxr-xr-x 116 root root       4096 Oct 30  2023 ..
drwxr-x---   4 root blukat     4096 Aug 16  2018 .

specifically, password is readable by the group specified (blukat_pwn), which is weird as most times it will be the owner's group / root. To find out more information I used the ID command to see my permissions as the user blukat:
blukat@pwnable:~$ id
uid=1104(blukat) gid=1104(blukat) groups=1104(blukat),1105(blukat_pwn)

This means that I am a part of 2 groups: blukat and blukat_pwn, which is the owner group of password. As the file is readable for the owner group blukat_pwn this means i should be accessible to see the content of password by its own:
blukat@pwnable:~$ cat password
cat: password: Permission denied
blukat@pwnable:~$ more password
cat: password: Permission denied

Right here I understood they are playing mind games with me and trying to make me schizophrenic, because at first it seems I cannot access password because the output of cat is the same as what you would expect from being denied, but the latter output shows that it is the actual content of password, meaning "cat: password: Permission denied" is the actual password


![[asdasdasdsadasdasd 1.png]]

blukat@pwnable:~$ ./blukat
guess the password!
cat: password: Permission denied
congrats! here is your flag: Pl3as_DonT_Miss_youR_GrouP_Perm!!