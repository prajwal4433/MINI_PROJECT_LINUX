# MINI_PROJECT_LINUX

Project Title: Chatting Application Using FIFO (Named Pipes) and IPC in C

Objective: To design and implement a two-way communication system between two users using FIFO (Named Pipes), enabling them to chat in real-time through inter-process communication in a Linux environment.

Project Information:

Communication Type: Inter-process communication (IPC) using named pipes (FIFOs).
Users: Two users (User 1 and User 2), each running the application in separate terminals.
Processes:
Parent process: Sends messages.
Child process: Receives messages (created using fork()).
FIFOs used:
/tmp/user1_fifo: For User 1 to receive messages.
/tmp/user2_fifo: For User 2 to receive messages.
Graceful Exit: Handles SIGINT (Ctrl+C) for proper cleanup of resources.
Duplex Communication: Each user writes to the other’s FIFO and reads from their own.


    /*  Chatting application using Fifo using Inter process communication mechanism. */


    
    #include<stdio.h>
    #include<stdlib.h>
    #include<string.h>
    #include<fcntl.h>
    #include<sys/stat.h>
    #include<sys/types.h>
    #include<unistd.h>
    #include<signal.h>
    #include<errno.h>

    #define FIFO_USER1 "/tmp/user1_fifo"
    #define FIFO_USER2 "/tmp/user2_fifo"

  
    int fd_send, fd_recv;
    pid_t child_pid;

    void cleanup()

    {

    if(fd_send>0)
            close(fd_send);
    if(fd_recv>0)
            close(fd_recv);
    if(child_pid>0)
            kill(child_pid, SIGTERM);
    }

    void handle_sigint(int sig)
      
    {

    cleanup();
    exit(0);
    }

    int main(int argc, char *argv[])
              
    {
    
    char message[100];
    char recv_message[100];
    
    if(argc != 2)
    {
        printf("Usage: %s <1 or 2>\n", argv[0]);
        return 1;
    }

    int user=atoi(argv[1]);
    if(user!=1 && user!=2)
    {
        printf("User must be either 1 or 2\n");
        return 1;
    }

    //Create FIFOs
    mkfifo(FIFO_USER1, 0666);
    mkfifo(FIFO_USER2, 0666);

    //Open pipes in the correct order to avoid deadlock
    if(user == 1)
    {
        printf("User 1: Waiting for User 2 to connect...\n");
        fd_send=open(FIFO_USER2, O_WRONLY);
        fd_recv=open(FIFO_USER1, O_RDONLY);
    }
    else
    {
        printf("User 2: Waiting for User 1 to connect...\n");
        fd_recv=open(FIFO_USER2, O_RDONLY);
        fd_send=open(FIFO_USER1, O_WRONLY);
    }

    printf("Connected! You can start chatting now.\n");

    signal(SIGINT, handle_sigint);

    child_pid=fork();

    if(child_pid==0)
    {
        //Child process handles receiving messages
        while(1)
        {
            int bytes_read=read(fd_recv, recv_message, sizeof(recv_message));
            if(bytes_read > 0)
            {
                printf("\nReceived: %s", recv_message);
                fflush(stdout);
            }
            else if (bytes_read == 0)
            {
                //Pipe closed by other end
                printf("\nOther user disconnected.\n");
                cleanup();
                exit(0);
            }
        }
    }
    else
    {
        //Parent process - handles sending messages
        while(1)
        {
            printf("You: ");
            fflush(stdout);  //Ensure prompt is displayed
            if(fgets(message, sizeof(message), stdin)==NULL)
            {
                //Handle Ctrl+D (EOF)
                printf("\n");
                break;
            }

           if(strcmp(message, "exit\n")==0)
           {
                cleanup();
                break;
            }

            write(fd_send, message, strlen(message)+1);
        }
    }

    cleanup();
    return 0;
}

