# Using-semaphores-in-interprocess-communication

https://docs.docker.com/get-started/02_our_app/
https://docs.docker.com/compose/gettingstarted/

## Communicating through Shared Memory with Semaphores in C

In this article, we will explore how to use shared memory and semaphores to facilitate communication between two programs in C. Shared memory allows multiple processes to access the same region of memory, while semaphores enable synchronization and control of access to shared resources. We will implement two programs: a sender and a receiver, which will communicate by exchanging messages through shared memory.

### Shared Memory

Shared memory is a mechanism that allows multiple processes to access the same memory region. In C, we can use the `shmget` function to create or access a shared memory segment. Here's an overview of the functions used in our programs:

- `shmget(key, size, flags)`: This function returns the identifier of the shared memory segment associated with the given key. If the shared memory segment does not exist, it creates one. The `size` parameter specifies the size of the shared memory segment in bytes, and `flags` determine the access permissions for the shared memory segment.

- `shmat(shmid, shmaddr, flags)`: This function attaches the shared memory segment specified by `shmid` to the address space of the calling process. The `shmaddr` parameter specifies the address at which to attach the shared memory segment. By passing `NULL`, the system chooses a suitable address. The `flags` parameter determines the attachment options.

- `shmdt(shmaddr)`: This function detaches the shared memory segment from the address space of the calling process, allowing other processes to access it. The `shmaddr` parameter specifies the address of the shared memory segment to detach.

- `shmctl(shmid, cmd, buf)`: This function performs control operations on the shared memory segment. For example, we can use it to remove the shared memory segment when it is no longer needed.

### Semaphores

Semaphores are synchronization primitives used to control access to shared resources. They provide a way to limit the number of processes accessing a resource simultaneously. In our programs, we will use a binary semaphore to allow or block access to the shared memory.

Here are the semaphore functions used in our programs:

- `sem_open(name, oflag, mode, value)`: This function opens or creates a named semaphore. The `name` parameter specifies the name of the semaphore, `oflag` determines the creation or opening options, `mode` specifies the access permissions, and `value` sets the initial value of the semaphore.

- `sem_wait(semaphore)`: This function decrements the value of the semaphore. If the value becomes negative, the calling process is blocked until it can decrement the semaphore.

- `sem_post(semaphore)`: This function increments the value of the semaphore, potentially unblocking a waiting process.

- `sem_close(semaphore)`: This function closes the semaphore.

- `sem_unlink(name)`: This function removes the named semaphore.

### Implementing the Sender and Receiver Programs

Now let's implement the sender and receiver programs that communicate through shared memory using semaphores.

#### Sender Program

```c
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <unistd.h>
#include <sys/ipc.h>
#include <sys/shm.h>
#include <semaphore.h>

#define SEM_NAME "/my_semaphore"

int main() {
    key_t key = ftok(".", 'S');
    int shmid = shmget(key, 1024, IPC_CREAT | 0666);
    char* shmessage = (char*)shmat(shmid, NULL, 0);

    sem_t* semaphore = sem_open(SEM_NAME, O_CREAT, 0666,

 1); // Create and initialize the semaphore
    if (semaphore == SEM_FAILED) {
        perror("sem_open");
        exit(1);
    }

    while (1) {
        sem_wait(semaphore); // Lock the semaphore

        memcpy(shmessage, "CIEPŁO", sizeof("CIEPŁO"));
        sleep(1);

        memcpy(shmessage, "ZIMNO", sizeof("ZIMNO"));
        sleep(1);

        sem_post(semaphore); // Unlock the semaphore
    }

    sem_close(semaphore); // Close the semaphore
    sem_unlink(SEM_NAME); // Remove the semaphore

    shmdt(shmessage);
    shmctl(shmid, IPC_RMID, NULL);

    return 0;
}
```

The sender program creates a shared memory segment using `shmget` and attaches it to the process using `shmat`. It also creates a binary semaphore using `sem_open` with the `O_CREAT` flag, which ensures that the semaphore is created if it does not exist. The semaphore is initialized with a value of 1 to allow access initially.

Inside the infinite loop, the program locks the semaphore using `sem_wait`, writes the message "CIEPŁO" to the shared memory using `memcpy`, sleeps for a second, writes the message "ZIMNO" to the shared memory, and sleeps again. After each write operation, the program unlocks the semaphore using `sem_post` to allow the receiver program to access the shared memory.

Finally, the program closes and unlinks the semaphore using `sem_close` and `sem_unlink`, respectively. It also detaches the shared memory using `shmdt` and removes the shared memory segment using `shmctl`.

#### Receiver Program

```c
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <unistd.h>
#include <sys/ipc.h>
#include <sys/shm.h>
#include <semaphore.h>

#define SEM_NAME "/my_semaphore"

int main() {
    key_t key = ftok(".", 'S');
    int shmid = shmget(key, 1024, IPC_CREAT | 0666);
    char* shmessage = (char*)shmat(shmid, NULL, 0);

    sem_t* semaphore = sem_open(SEM_NAME, 0); // Open the existing semaphore
    if (semaphore == SEM_FAILED) {
        perror("sem_open");
        exit(1);
    }

    while (1) {
        sem_wait(semaphore); // Lock the semaphore

        if (strcmp(shmessage, "CIEPŁO") == 0) {
            printf("Odebrano: CIEPŁO\n");
            sleep(1);
        }
        else if (strcmp(shmessage, "ZIMNO") == 0) {
            printf("Odebrano: ZIMNO\n");
            sleep(1);
        }
        else {
            printf("Błąd\n");
            sleep(1);
        }

        sem_post(semaphore); // Unlock the semaphore
    }

    sem_close(semaphore); // Close the semaphore

    shmdt(shmessage);
}
```

The receiver program accesses the shared memory and semaphore using similar code as the sender program. Instead of creating the semaphore, it opens the existing semaphore using `sem_open` without the `O_CREAT` flag. If the semaphore does not exist, the program prints an error message and exits.

Inside the infinite loop, the program locks the semaphore using `sem_wait`, reads the shared memory, and checks if the message is "CIEPŁO" or "ZIMNO". It then prints the appropriate message and

 sleeps for a second. If the message is neither "CIEPŁO" nor "ZIMNO", the program prints an error message.

After processing the message, the program unlocks the semaphore using `sem_post` to allow the sender program to write to the shared memory.

Finally, the program closes the semaphore using `sem_close` and detaches the shared memory using `shmdt`.

### Compiling the Programs

To compile the sender and receiver programs, we need to link the `-lrt` library to include support for POSIX semaphores. Here's an example of how to compile the programs using `gcc`:

```
gcc sender.c -o sender -lrt
gcc receiver.c -o receiver -lrt
```

Make sure to run these commands in your terminal to generate the executable files `sender` and `receiver`.

### Conclusion

In this article, we learned how to use shared memory and semaphores to facilitate communication between processes in C. By creating a shared memory segment and using a binary semaphore, we established a synchronized mechanism for exchanging messages between the sender and receiver programs. This approach can be extended to facilitate communication between multiple processes and enable more complex interactions.
