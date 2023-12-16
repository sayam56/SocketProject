/*
member1 : Ali Iktider | 110122251 | Section 2
member2: Ashik Al Habib | 110107722 | Section 2
*/

#include <netinet/in.h>
#include <stdio.h>
#include <stdlib.h>
#include <sys/socket.h> //for socket APIs
#include <sys/types.h>
#include <sys/wait.h>
#include <string.h>
#include <time.h>
#include <sys/stat.h>
#include <dirent.h>
#include <fcntl.h>
#include <unistd.h>
#include <arpa/inet.h>

#define _POSIX_SOURCE

void createTarDir() {
    const char *directoryName = "/home/iktider/Desktop/f23project"; // Change this to your desired directory name

    // Create the directory
    if (mkdir(directoryName, S_IRWXU | S_IRWXG | S_IROTH | S_IXOTH) == 0) {
        //printf("Directory created successfully: %s\n", directoryName);
    } else {
        perror("mkdir");
        exit(EXIT_FAILURE);
    }
}

//helpers here

void searchDirectory(int client_sd, const char *search_path, const char *filename) {
    DIR *dir = opendir(search_path);
    if (dir == NULL) {
        perror("opendir");
        return;
    }

    struct dirent *entry;
    int file_found = 0; // Flag to check if the file is found
    while ((entry = readdir(dir)) != NULL) {
        if (strcmp(entry->d_name, ".") == 0 || strcmp(entry->d_name, "..") == 0) //avoiding the inifite loop to root and current
            continue;

        char filepath[1024];
        snprintf(filepath, sizeof(filepath), "%s/%s", search_path, entry->d_name);

        struct stat file_stat;
        if (stat(filepath, &file_stat) == 0) {
            if (S_ISDIR(file_stat.st_mode)) {
                // recursively checking inside any sub directgories
                searchDirectory(client_sd, filepath, filename);
            } else if (S_ISREG(file_stat.st_mode) && strcmp(entry->d_name, filename) == 0) {
                // if found
                char response[2048];

                // Extract file permissions
                char permissions[4];
                permissions[0] = (file_stat.st_mode & S_IRUSR) ? 'r' : '-';
                permissions[1] = (file_stat.st_mode & S_IWUSR) ? 'w' : '-';
                permissions[2] = (file_stat.st_mode & S_IXUSR) ? 'x' : '-';
                permissions[3] = '\0';

                // Convert modification time to string using creation time
                char *date_str = ctime(&file_stat.st_ctime);

                snprintf(response, sizeof(response),
                         "File: %s\nSize: %ld bytes\nPermissions: %s\nDate Created: %s",
                         filename, file_stat.st_size, permissions, date_str);
                write(client_sd, response, strlen(response) + 1);
                file_found = 1;
                break; // Stop searching once the file is found
            }
        }
    }

    closedir(dir);

    // If the file is not found, send a "File not found" message
    if (!file_found) {
        const char *not_found_msg = "File not found";
        write(client_sd, not_found_msg, strlen(not_found_msg) + 1);
    }
}


void appendFilePath(char **fileList, const char *filePath)
{
    size_t currentSize = strlen(*fileList);
    size_t filePathSize = strlen(filePath);
    char *newList = realloc(*fileList, currentSize + filePathSize + 2);
    if (newList == NULL)
    {
        perror("realloc");
        exit(EXIT_FAILURE);
    }
    *fileList = newList;
    strcat(*fileList, filePath);
    strcat(*fileList, "\n");
}

void findsizerange(unsigned long long minSize, unsigned long long maxSize, char **fileList, const char *searchPath)
{
    DIR *dir = opendir(searchPath);
    if (dir == NULL)
    {
        perror("opendir");
        return;
    }

    struct dirent *entry;
    while ((entry = readdir(dir)) != NULL)
    {
        if (strcmp(entry->d_name, ".") == 0 || strcmp(entry->d_name, "..") == 0)
        {
            continue;
        }

        char filePath[256];
        snprintf(filePath, sizeof(filePath), "%s/%s", searchPath, entry->d_name);

        struct stat fileStat;
        if (stat(filePath, &fileStat) == 0)
        {
            if (S_ISREG(fileStat.st_mode) && fileStat.st_size >= minSize && fileStat.st_size <= maxSize)
            {
                appendFilePath(fileList, filePath);
            }
            else if (S_ISDIR(fileStat.st_mode))
            {
                // recursive call to find files inside sub dir
                findsizerange(minSize, maxSize, fileList, filePath);
            }
        }
        else
        {
            //
        }
    }

    closedir(dir);
}

void searchforFiles(char *file_list_str, const char *path, const char *filename, int *file_count)
{
    DIR *dir = opendir(path);
    if (dir == NULL)
    {
        perror("opendir");
        return;
    }

    struct dirent *entry;
    while ((entry = readdir(dir)) != NULL)
    {
        if (strcmp(entry->d_name, ".") == 0 || strcmp(entry->d_name, "..") == 0)
        {
            continue;
        }

        char full_path[1024];
        snprintf(full_path, sizeof(full_path), "%s/%s", path, entry->d_name);

        struct stat entry_stat;
        if (stat(full_path, &entry_stat) != 0)
        {
            continue;
        }

        if (S_ISDIR(entry_stat.st_mode))
        {
            searchforFiles(file_list_str, full_path, filename, file_count);
        }
        else if (S_ISREG(entry_stat.st_mode) && strcmp(entry->d_name, filename) == 0)
        {
            // concat file with full_path
            strcat(file_list_str, full_path);
            // Insert space in between
            strcat(file_list_str, " ");
            (*file_count)++;
        }
    }

    closedir(dir);
}

// Used for getft command
void filesearchWrite(FILE *file_list, const char *path, const char *extensions)
{
    DIR *dir = opendir(path);
    if (dir == NULL)
    {
        perror("opendir");
        return;
    }
    //printf("extensions passed: %s\n", extensions);

    struct dirent *entry;
    while ((entry = readdir(dir)) != NULL)
    {
        if (strcmp(entry->d_name, ".") == 0 || strcmp(entry->d_name, "..") == 0)
        {
            continue;
        }

        char full_path[1024];
        snprintf(full_path, sizeof(full_path), "%s/%s", path, entry->d_name);

        struct stat entry_stat;
        if (stat(full_path, &entry_stat) != 0)
        {
            perror("stat");
            continue;
        }

        if (S_ISDIR(entry_stat.st_mode))
        {
            filesearchWrite(file_list, full_path, extensions);
        }
        else
        {
            if (S_ISREG(entry_stat.st_mode))
            {
                char *ext = strrchr(entry->d_name, '.');
                if (ext != NULL && strstr(extensions, ext))
                {
                    fprintf(file_list, "%s\n", full_path);
                }
            }
        }
    }

    closedir(dir);
}

// commands from here

// sayam: works fine
// =======getfz size1 size2===========
void getfz(int client_sd, char buff1[])
{

    unsigned long long minSize, maxSize;
    char cwd[256];

    const char *home_dir = getenv("HOME");

    // returns the number of fields that were successfully converted and assigned
    int assigned = sscanf(buff1 + 9, "%llu %llu", &minSize, &maxSize);

    // validation for client input
    if (assigned >= 2 && minSize <= maxSize)
    {
        // only perform these operations if command format is accurate
        sscanf(buff1 + 9, "%llu %llu", &minSize, &maxSize);
        printf("Requested size range: %llu - %llu\n", minSize, maxSize);

        char *fileList = (char *)malloc(1); // Start with an empty string
        if (fileList == NULL)
        {
            perror("malloc");
            exit(EXIT_FAILURE);
        }
        fileList[0] = '\0';

        findsizerange(minSize, maxSize, &fileList, home_dir);

        if (fileList[0] == '\0')
        {
            // No files found, send a message to the client
            char *noFileMsg = "No file found";
            write(client_sd, noFileMsg, strlen(noFileMsg) + 1);

            free(fileList);
            return;
        }

        // Create a temporary file to store the list of files
        char tmpFilePath[] = "/tmp/file_list.txt";
        int tmpFile = open(tmpFilePath, O_CREAT | O_WRONLY, 0644);
        if (tmpFile == -1)
        {
            perror("open");
            free(fileList);
            exit(EXIT_FAILURE);
        }
        write(tmpFile, fileList, strlen(fileList));
        close(tmpFile);

        // Create tar.gz archive
        pid_t pid = fork();
        if (pid == 0)
        {
            // in the child process
            chdir(home_dir); // Change to home directory

            // Create the tar.gz archive
            char *tmpFilePath = "/tmp/file_list.txt";
            execlp("tar", "tar", "-czvf", "f23project/temp.tar.gz", "-T", tmpFilePath, NULL);

            perror("execlp");
            exit(EXIT_FAILURE);
        }
        else if (pid > 0)
        {
            wait(NULL); // wait for child process to finish
        }
        else
        {
            perror("fork");
            free(fileList);
            exit(EXIT_FAILURE);
        }

        char *msg = "Finished tarring files";
        write(client_sd, msg, strlen(msg) + 1);

        // Delete the temporary file
        if (unlink(tmpFilePath) != 0)
        {
            perror("unlink");
        }

        free(fileList);
    }
    else
    {
        char *msg = "Usage: getfz size1 size2";

        write(client_sd, msg, 50);
    }
}

// sayam: Works fine, Date issue fixed
void getfn(int client_sd, char buff1[])
{

    //printf("Buff1 is : %s\n", buff1+5);
    const char *filename = buff1 + 6; // Extract the filename from the command+space
    printf("Requested file: %s\n", filename);

    // Get the user's home directory
    // const char *home_dir = getenv("HOME");
    char cwd[256];

    const char *home_dir = getenv("HOME");
    printf("Searching directory tree rooted at: %s\n", home_dir);

    // Recursively search the home directory for the file
    searchDirectory(client_sd, home_dir, filename);

    // File not found
    char *msg = "File not found";
    write(client_sd, msg, 100);
}

int countExtensions(const char *buffer) {
    char bufferCopy[256];  // Adjust the size as needed
    strcpy(bufferCopy, buffer);

    // Tokenize the buffer based on spaces
    char *token = strtok(bufferCopy, " ");
    int count = 0;

    // Count the number of tokens (extensions)
    while (token != NULL) {
        count++;
        token = strtok(NULL, " ");
    }

    return count;
}

//sayam: fixed extension issue
//====== getft <extension list> max 3 extentions allowed ====
void getft(int client_sd, char buff1[]) {
    char cwd[256];

    int num_extensions = countExtensions(buff1);
    num_extensions-=1;

    //printf("NUmber of extensions: %d\n", num_extensions);

    const char *extension_list = buff1 + 6; // Extract the extension list from the command


    //printf("extension array before tok: %s",extension_list);

    /*
    // Tokenize based on space
    char *ext = strtok((char *)extension_list, " ");

    // Count the number of extensions
    int num_extensions = 0;
    while (ext != NULL) {
        if (strlen(ext) > 0) {
            num_extensions++;
            //printf("Extension: %s\n", ext);
        }
        ext = strtok(NULL, " ");
    }

    printf("Number of extensions: %d\n",num_extensions);
    printf("extension array: %s",extension_arg);

    */
    // Check the count after tokenization
    if (num_extensions == 0 || num_extensions > 3) {
        const char *usage_msg = "Usage: getft <extension list> minimum 1 or maximum 3 extensions are allowed";
        send(client_sd, usage_msg, strlen(usage_msg), 0);
        return;
    }

    const char *home_dir = getenv("HOME");

    FILE *file_list = fopen("file_list.txt", "w");
    if (file_list == NULL) {
        perror("fopen");
        return;
    }

    // Recursively search for files with the specified extensions in the home directory
    filesearchWrite(file_list, home_dir, extension_list);

    fclose(file_list);

    // Check if any files were found
    FILE *check_file_list = fopen("file_list.txt", "r");
    if (check_file_list == NULL) {
        const char *no_file_msg = "No file found";
        send(client_sd, no_file_msg, strlen(no_file_msg), 0);
        return;
    }
    fclose(check_file_list);

    // Create a tar.gz archive from the file list
    char tar_command[1024];
    snprintf(tar_command, sizeof(tar_command), "tar -czf f23project/temp.tar.gz -T file_list.txt");

    system(tar_command);

    // Send the tar.gz archive to the client
    FILE *tar_file = fopen("f23project/temp.tar.gz", "rb");
    if (tar_file) {
        fseek(tar_file, 0, SEEK_END);
        long tar_size = ftell(tar_file);
        rewind(tar_file);

        char *tar_buffer = (char *)malloc(tar_size);
        fread(tar_buffer, 1, tar_size, tar_file);
        fclose(tar_file);

        send(client_sd, tar_buffer, tar_size, 0);

        free(tar_buffer);
    } else {
        const char *error_msg = "Error creating tar.gz file";
        send(client_sd, error_msg, strlen(error_msg), 0);
    }

    const char *msg = "Finished tarring files.";
    write(client_sd, msg, strlen(msg));

    // Clean up
    remove("file_list.txt");
}

//sayam: both getfdb and getfda does not work properly.
// fixed getfdb and fda
// getfdb command for date
void getfdb(int client_sd, const char* date)
{
    char cwd[256];
    const char *home_dir = getenv("HOME");

    char command[1024];
    char fileList[1024];

    // Construct the find command
    snprintf(command, sizeof(command), "find %s -type f ! -newermt %s", home_dir, date);

    // Open a pipe to the command
    FILE *fp = popen(command, "r");
    if (fp == NULL) {
        perror("popen");
        exit(EXIT_FAILURE);
    }

    // Read the output of the command into the fileList array
    size_t bytesRead = fread(fileList, 1, sizeof(fileList) - 1, fp);
    if (bytesRead == 0) {
        if (feof(fp)) {
            printf("No files found.\n");
        } else {
            perror("fread");
        }
    } else {
        // Null-terminate the array
        fileList[bytesRead] = '\0';

        // Print or manipulate the fileList as needed
        printf("Files:\n%s\n", fileList);
    }

    // Close the pipe
    pclose(fp);

    // Create a temporary text file to store the list of files
    char tmpFilePath[] = "/tmp/file_list.txt";
    int tmpFile = open(tmpFilePath, O_CREAT | O_WRONLY, 0644);
    if (tmpFile == -1)
    {
        perror("open");
        return;
    }
    write(tmpFile, fileList, strlen(fileList));
    close(tmpFile);

    // Create tar.gz archive of the files list
    pid_t pid = fork();
    if (pid == 0)
    {
        chdir(home_dir); // Change to the home directory
        execlp("tar", "tar", "-czvf", "f23project/temp.tar.gz", "-T", tmpFilePath, NULL);
        perror("execlp");
        exit(EXIT_FAILURE);
    }
    else if (pid > 0)
    {
        wait(NULL);
    }
    else
    {
        perror("fork");
        return;
    }

    printf("Tar.gz archive created: temp.tar.gz\n");

    // Delete the temporary file
    if (unlink(tmpFilePath) != 0)
    {
        perror("unlink");
    }

    // Send the tar.gz archive to the client
    FILE *tar_file = fopen("temp.tar.gz", "rb");
    if (tar_file)
    {
        fseek(tar_file, 0, SEEK_END);
        long tar_size = ftell(tar_file);
        rewind(tar_file);

        char *tar_buffer = (char *)malloc(tar_size);
        fread(tar_buffer, 1, tar_size, tar_file);
        fclose(tar_file);

        send(client_sd, tar_buffer, tar_size, 0);

        free(tar_buffer);
    }
    else
    {
        const char *error_msg = "Error creating tar.gz file";
        send(client_sd, error_msg, strlen(error_msg), 0);
    }
}


// getfda command for date
void getfda(int client_sd, char date[])
{
    char cwd[256];
    const char *home_dir = getenv("HOME");

    char command[1024];
    char fileList[1024];

    // Construct the find command
    snprintf(command, sizeof(command), "find %s -type f -newermt %s", home_dir, date);

    // Open a pipe to the command
    FILE *fp = popen(command, "r");
    if (fp == NULL) {
        perror("popen");
        exit(EXIT_FAILURE);
    }

    // Read the output of the command into the fileList array
    size_t bytesRead = fread(fileList, 1, sizeof(fileList) - 1, fp);
    if (bytesRead == 0) {
        if (feof(fp)) {
            printf("No files found.\n");
        } else {
            perror("fread");
        }
    } else {
        // Null-terminate the array
        fileList[bytesRead] = '\0';

        // Print or manipulate the fileList as needed
        printf("Files:\n%s\n", fileList);
    }

    // Close the pipe
    pclose(fp);

    // Create a temporary text file to store the list of files
    char tmpFilePath[] = "/tmp/file_list.txt";
    int tmpFile = open(tmpFilePath, O_CREAT | O_WRONLY, 0644);
    if (tmpFile == -1)
    {
        perror("open");
        return;
    }
    write(tmpFile, fileList, strlen(fileList));
    close(tmpFile);

    // Create tar.gz archive of the files list
    pid_t pid = fork();
    if (pid == 0)
    {
        chdir(home_dir); // Change to the home directory
        execlp("tar", "tar", "-czvf", "f23project/temp.tar.gz", "-T", tmpFilePath, NULL);
        perror("execlp");
        exit(EXIT_FAILURE);
    }
    else if (pid > 0)
    {
        wait(NULL);
    }
    else
    {
        perror("fork");
        return;
    }

    printf("Tar.gz archive created: temp.tar.gz\n");

    // Delete the temporary file
    if (unlink(tmpFilePath) != 0)
    {
        perror("unlink");
    }

    // Send the tar.gz archive to the client
    FILE *tar_file = fopen("temp.tar.gz", "rb");
    if (tar_file)
    {
        fseek(tar_file, 0, SEEK_END);
        long tar_size = ftell(tar_file);
        rewind(tar_file);

        char *tar_buffer = (char *)malloc(tar_size);
        fread(tar_buffer, 1, tar_size, tar_file);
        fclose(tar_file);

        send(client_sd, tar_buffer, tar_size, 0);

        free(tar_buffer);
    }
    else
    {
        const char *error_msg = "Error creating tar.gz file";
        send(client_sd, error_msg, strlen(error_msg), 0);
    }
}

// finishing client commands


//pclientrequest
//server is processing client requests from here
void pclientrequest(int client_sd)
{

    char buff1[1024];
    ssize_t bytes_read = read(client_sd, buff1, sizeof(buff1) - 1);

    if (bytes_read <= 0)
    {
        perror("read");
        close(client_sd);
        return;
    }

    buff1[bytes_read] = '\0';

    if (strcmp(buff1, "quitc") == 0)
    {
        printf("Client Process Terminated!\n");
    }
    // compare first 5 characters of buff1 with "getfz"
    else if (strncmp(buff1, "getfz ", 5) == 0)
    {
        getfz(client_sd, buff1);
    }
    else if (strncmp(buff1, "getfn ", 5) == 0)
    {
        // code here
        getfn(client_sd, buff1);
    }
    else if (strncmp(buff1, "getft ", 5) == 0)
    {
        getft(client_sd, buff1);
    }
    // else if (strncmp(buff1, "getdirf ", 7) == 0)
    // {
    //     getdirf(client_sd, buff1);
    // }
    else if (strncmp(buff1, "getfdb ", 6) == 0)
    {
        char date[20];
        sscanf(buff1 + 7, "%s", date);
        printf("Requested files created on or before: %s\n", date);
        getfdb(client_sd, date);
    }
    else if (strncmp(buff1, "getfda ", 6) == 0)
    {
        char date[20];
        sscanf(buff1 + 7, "%s", date);
        printf("Requested files created on or after: %s\n", date);
        getfda(client_sd, date);
    }
    else
    {
        // Unknown command
        const char *error_msg = "Unknown command";
        send(client_sd, error_msg, strlen(error_msg), 0);
    }
}

int main(int argc, char *argv[])
{
    char *myTime;
    time_t currentUnixTime; // time.h
    int lis_sd, con_sd, portNumber;
    socklen_t len;
    struct sockaddr_in servAdd;

    if (argc != 2)
    {
        fprintf(stderr, "Call model: %s <Port#>\n", argv[0]);
        exit(0);
    }
    // socket()
    if ((lis_sd = socket(AF_INET, SOCK_STREAM, 0)) < 0)
    {
        fprintf(stderr, "Could not create socket\n");
        exit(1);
    }

    servAdd.sin_family = AF_INET;
    servAdd.sin_addr.s_addr = htonl(INADDR_ANY);
    sscanf(argv[1], "%d", &portNumber);
    servAdd.sin_port = htons((uint16_t)portNumber);

    // bind
    bind(lis_sd, (struct sockaddr *)&servAdd, sizeof(servAdd));

    struct sockaddr_in serverAddress;
    socklen_t addrLen = sizeof(serverAddress);

    // Get the address information of the bound socket
    if (getsockname(lis_sd, (struct sockaddr *)&serverAddress, &addrLen) == -1)
    {
        perror("getsockname");
        exit(1);
    }

    // Convert binary IPv4 address to human-readable format
    char ipAddress[INET_ADDRSTRLEN];
    if (inet_ntop(AF_INET, &(serverAddress.sin_addr), ipAddress, INET_ADDRSTRLEN) == NULL)
    {
        perror("inet_ntop");
        exit(1);
    }

    printf("Mirror IP Address: %s\n", ipAddress);

    // listen

    listen(lis_sd, 4);

    int numChildren = 1; // Count of child processes
    while (1)
    {

        printf("Mirror listening on port: %d...\n", portNumber);

        con_sd = accept(lis_sd, (struct sockaddr *)NULL, NULL); // accept()
        if (con_sd < 0)
        {
            perror("accept");
            continue;
        }


        int pid = fork();
        if (pid == 0)
        {
            // Child process
            // Close listening socket in child
            close(lis_sd);

            pclientrequest(con_sd);

            close(con_sd);

            // Child process exits
            exit(0);
        }
        else if (pid > 0)
        {
            // Parent process
            close(con_sd); // Close connection socket in parent
            numChildren++;
        }
        else
        {
            printf("error forking");
        }
    }

    int status;
    pid_t finished_child = wait(&status); // Wait for any child process to finish
    if (finished_child > 0)
    {
        numChildren--;
        printf("Child process with PID %d has finished.\n", finished_child);
    }
}
