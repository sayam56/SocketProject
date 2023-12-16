
/*
member1 : Ali Iktider | 110122251 | Section 2
member2: Ashik Al Habib | 110107722 | Section 2
*/


#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <sys/socket.h>
#include <sys/types.h>
#include <netinet/in.h>

int main(int argc, char *argv[])
{
    char cmd[100]; // Buffer size for client command
    int client_sd, portNum;
    socklen_t len;
    struct sockaddr_in servAdd;

    if (argc != 3)
    {
        // Checking correct command usage
        printf("Usage:%s <IP> <PortNum>\n", argv[0]);
        exit(0);
    }

    // Create socket
    if ((client_sd = socket(AF_INET, SOCK_STREAM, 0)) < 0)
    {
        fprintf(stderr, "Socket Error!\n");
        exit(1);
    }

    // Setting server address
    servAdd.sin_family = AF_INET;
    sscanf(argv[2], "%d", &portNum);
    servAdd.sin_port = htons((uint16_t)portNum);

    // Converting IP add. from txt to binary
    if (inet_pton(AF_INET, argv[1], &servAdd.sin_addr) < 0)
    {
        fprintf(stderr, "inet_pton() has failed!\n");
        exit(2);
    }

    // Connect to the server
    if (connect(client_sd, (struct sockaddrnano *)&servAdd, sizeof(servAdd)) < 0)
    {
        fprintf(stderr, "Connection Error!\n");
        exit(3);
    }

    char buff[50];
    printf("\n Enter command for Server: \n");
    gets(&buff);
    printf("Sending the Command: %s\n", buff);
    write(client_sd, buff, 100);

    // Reading server's response
    if (read(client_sd, cmd, 100) < 0)
    {
        fprintf(stderr, "Error Reading File!\n");
        exit(3);
    }

    fprintf(stderr, "%s\n", cmd);

    // Close client socket
    close(client_sd);

    exit(0);
}