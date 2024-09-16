// Includem bibliotecile necesare
#include <stdio.h>
#include <stdlib.h>
#include <unistd.h>
#include <sys/wait.h>
#include <ctype.h>

// Definim dimensiunea maximă a bufferului
#define MAX_SIZE 1024

// Funcție care calculează statistica caracterelor dintr-un șir
void calculate_statistics(char *str, int *digits, int *upper, int *lower) {
    // Inițializăm contoarele cu zero
    *digits = 0;
    *upper = 0;
    *lower = 0;

    // Parcurgem șirul caracter cu caracter
    for (int i = 0; str[i] != '\0'; i++) {
        // Verificăm dacă este o cifră
        if (isdigit(str[i])) {
            // Incrementăm numărul de cifre
            (*digits)++;
        }
        // Verificăm dacă este o literă mare
        else if (isupper(str[i])) {
            // Incrementăm numărul de litere mari
            (*upper)++;
        }
        // Verificăm dacă este o literă mică
        else if (islower(str[i])) {
            // Incrementăm numărul de litere mici
            (*lower)++;
        }
    }
}

// Funcția principală a programului
int main(int argc, char *argv[]) {
    // Verificăm dacă a fost furnizat un fișier ca argument
    if (argc != 2) {
        printf("Utilizare: %s <fisier>\n", argv[0]);
        exit(1);
    }

    // Deschidem fișierul pentru citire
    FILE *fp = fopen(argv[1], "r");
    if (fp == NULL) {
        perror("Eroare la deschiderea fișierului");
        exit(2);
    }

    // Declaram două pipe-uri, unul pentru trimiterea datelor de la părinte la fiu
    // și altul pentru trimiterea datelor de la fiu la părinte
    int pipe1[2], pipe2[2];

    // Creăm pipe-urile și verificăm dacă au fost create cu succes
    if (pipe(pipe1) == -1 || pipe(pipe2) == -1) {
        perror("Eroare la crearea pipe-urilor");
        exit(3);
    }

    // Creăm un proces fiu și verificăm dacă a fost creat cu succes
    pid_t pid = fork();
    if (pid == -1) {
        perror("Eroare la crearea procesului fiu");
        exit(4);
    }

    // Dacă suntem în procesul fiu
    if (pid == 0) {
        // Închidem capătul de scriere al primului pipe
        close(pipe1[1]);
        // Închidem capătul de citire al celui de-al doilea pipe
        close(pipe2[0]);

        // Declaram variabile pentru a stoca statistica caracterelor
        int digits, upper, lower;
        // Declaram un buffer pentru a citi datele din pipe
        char buffer[MAX_SIZE];

        // Citim datele din primul pipe până când nu mai avem nimic de citit
        while (read(pipe1[0], buffer, MAX_SIZE) > 0) {
            // Calculăm statistica caracterelor din buffer
            calculate_statistics(buffer, &digits, &upper, &lower);
            // Scriem statistica în cel de-al doilea pipe
            write(pipe2[1], &digits, sizeof(int));
            write(pipe2[1], &upper, sizeof(int));
            write(pipe2[1], &lower, sizeof(int));
        }

        // Închidem capătul de citire al primului pipe
        close(pipe1[0]);
        // Închidem capătul de scriere al celui de-al doilea pipe
        close(pipe2[1]);

        // Terminăm procesul fiu
        exit(0);
    }
    // Dacă suntem în procesul părinte
    else {
        // Închidem capătul de citire al primului pipe
        close(pipe1[0]);
        // Închidem capătul de scriere al celui de-al doilea pipe
        close(pipe2[1]);

        // Declaram variabile pentru a stoca statistica caracterelor
        int digits, upper, lower;
        // Declaram un buffer pentru a citi datele din fișier
        char buffer[MAX_SIZE];

        // Citim datele din fișier până când nu mai avem nimic de citit
        while (fgets(buffer, MAX_SIZE, fp) != NULL) {
            // Scriem datele în primul pipe
            write(pipe1[1], buffer, MAX_SIZE);
            // Citim statistica din cel de-al doilea pipe
            read(pipe2[0], &digits, sizeof(int));
            read(pipe2[0], &upper, sizeof(int));
            read(pipe2[0], &lower, sizeof(int));
            // Afișăm statistica
            printf("Cifre: %d\n", digits);
            printf("Litere mari: %d\n", upper);
            printf("Litere mici: %d\n", lower);
        }

        // Închidem fișierul
        fclose(fp);
        // Închidem capătul de scriere al primului pipe
        close(pipe1[1]);
        // Închidem capătul de citire al celui de-al doilea pipe
        close(pipe2[0]);

        // Așteptăm terminarea procesului fiu
        wait(NULL);

        // Terminăm procesul părinte
        exit(0);
    }

    return 0;
}

This C program uses the fork, pipe, read, write, stat, close, and wait system calls to create and communicate between two processes. 
The parent process reads a text file containing numbers, uppercase letters, lowercase letters and sends it through a pipe to the child process.
The child process performs a statistic with the number of characters received from each category and finally sends the statistic via a pipe to the parent process, which will display it.
