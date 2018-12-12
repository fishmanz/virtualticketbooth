//Zachary Fishman 2016

#include <unistd.h>
#include <errno.h>
#include <assert.h>

#include <sys/types.h>
#include <sys/stat.h>
#include <fcntl.h>

#include <stdio.h>
#include <stdlib.h>
#include <string.h>

#include <pthread.h>

#include "oshow.h"

#define handle_error(msg) \
        do { perror(msg); exit(EXIT_FAILURE); } while (0)

#define handle_error_en(en, msg) \
        do { errno = en; perror(msg); exit(EXIT_FAILURE); } while (0)


// there are 10 kiosks represented as named pipe kiosk[0-9].
// opening named pipes. 
int open_kiosk(int n, int* fd_client, int* fd_server)
{
    int fd_c, fd_s;
    char name[16];
    
    if (n < 0 || n > 9) handle_error_en(ERANGE, "kiosk number");

    sprintf(name, "kiosk-client%d", n);
    fd_c = open(name, O_RDONLY);    // this will block until other end opens
    if (fd_c == -1) handle_error("open (read)  failed");
    sprintf(name, "kiosk-server%d", n);
    fd_s = open(name, O_WRONLY);
    if (fd_s == -1) handle_error("open (write) failed");

    *fd_client = fd_c;
    *fd_server = fd_s;

    return 0;
}

struct thread_info {
    int id;
    int fd_client;
    int fd_server;
};

#define TOTAL_FLOOR (100)
#define TOTAL_MEZZANINE (80)
#define TOTAL_BALCONY (120)
#define TOTAL_BOX (30)

#define FLO (0)
#define MEZ (1)
#define BAL (2)
#define BOX (3)
#define NUM_locationS (4)

// our shared object
struct _tickets
{
    int total[NUM_locationS];
    int price[NUM_locationS];
    int reserved[NUM_locationS];
    int sequence_number; //does not ask sequnce to note seats or be w/o gaps
    pthread_cond_t CV[NUM_locationS];// are notified if corresponding "total" or "reserved" is changed
    pthread_mutex_t lock;
} tickets;

void tickets_init(void)
{
    memset(&tickets, 0, sizeof(tickets));
    tickets.total[FLO] = TOTAL_FLOOR;
    tickets.total[MEZ] = TOTAL_MEZZANINE;
    tickets.total[BAL] = TOTAL_BALCONY;
    tickets.total[BOX] = TOTAL_BOX;

    tickets.price[FLO] = 150;
    tickets.price[MEZ] = 130;
    tickets.price[BAL] = 90;
    tickets.price[BOX] = 110;

    tickets.sequence_number = 0;

    pthread_mutex_init(&tickets.lock, NULL);
    int i;
    for (i = 0; i < NUM_locationS; i++)
        pthread_cond_init(&tickets.CV[i], NULL);
}
/*
 * read into buf a 4-byte message from pipe
 */
ssize_t read_msg(struct thread_info* ti, int32_t* buf) {
    ssize_t n;
    n = read(ti->fd_client, buf, 4);
    if (n == 0) {
	return n;   // pipe closed
    }
    if (n != 4) handle_error("message (read) failed");
    return n;
}

/*
 * write to pipe 4-byte message val
 */
ssize_t write_msg(struct thread_info* ti, int32_t val) {
    ssize_t n;
    n = write(ti->fd_server, &val, 4);
    if (n != 4) handle_error("message (write) failed");
    return n;
}

/*
 * main loop exit condition check
 * returns 1 if all tickets are sold out and all pipe closed.
 */
int check_all_sold_out(void) {
    pthread_mutex_lock(&tickets.lock);
    int count = 0;
    int i;
    
    for (i = 0; i < NUM_locationS; i++) //Loops through all possible seat locations
    {
        count += tickets.total[i];
        count += tickets.reserved[i];
    }
    pthread_mutex_unlock(&tickets.lock);
    return count == 0;
}

//extra functions I added to make everything more streamlined and less messy, defintions are beneath main
int reserve_for(int location, int *sequence_number, int *price);
int reserve_ticket(int location, int *sequence_number, int *price);
void finalize_ticket(int location);
void release_reserved_ticket(int location);
void get_quote(struct thread_info* ti, int location);

void* kiosk_main(void* arg)
{
    struct thread_info* ti = (struct thread_info*)arg;
    int res;
    int msg;

    res = open_kiosk(ti->id, &ti->fd_client, &ti->fd_server);
    if (res) handle_error("open_kiosk failed");
    
    while (1) {
	msg = 0;
	if (read_msg(ti, &msg) == 0) {
	    if (!check_all_sold_out()) {
		printf("(%d) operation failure.\n", ti->id);
	    }
	    break;
	}

	switch (msg) {
	case QUOTE_ME_FLOOR:
	case QUOTE_ME_MEZZANINE:
	case QUOTE_ME_BALCONY:
	case QUOTE_ME_BOX:
	{
        get_quote(ti, msg - 1); //get_quote checks which type of seat, then calls other functions to reserve or unreserve
	}
	break;
	default:
	    fprintf(stderr, "(%d) incorrect message seq.\n", ti->id);
	    break;
	}

    }
    return ti;
}

#define NTHREADS (10)
int main(int argc, char** argv)
{
    int s, i;
    void* res;

    pthread_t thread[NTHREADS];

    tickets_init();
    for (i = 0; i < NTHREADS; i++) {
	struct thread_info* ti = malloc(sizeof(*ti));
	if (!ti) handle_error_en(ENOMEM, "malloc failed");
	ti->id = i;
	s = pthread_create(&thread[i], NULL, &kiosk_main, ti);
	if (s) handle_error_en(s, "pthread_create: ");
    }

    for (i = 0; i < NTHREADS; i++) {
	s = pthread_join(thread[i], &res);
	if (s) perror("pthread_join:");
	printf("thread %d joined\n", i);
    }
    
    printf("main done\n");
    return 0;
}

int reserve_for(int location, int *sequence_number, int *price)
{
    tickets.reserved[location]++;
    tickets.total[location]--;
    *sequence_number = ++tickets.sequence_number;
    *price = tickets.price[location];
    return 1;
 }
 
int reserve_ticket(int location, int *sequence_number, int *price)
{
    pthread_mutex_lock(&tickets.lock);
 
    int res = 0;
    if (tickets.total[location] > 0)
        res = reserve_for(location, sequence_number, price);
    else
    {
        if (tickets.reserved[location] > 0)
        {
            while (tickets.total[location] == 0 &&
                   tickets.reserved[location] > 0)
                pthread_cond_wait(&tickets.CV[location],
                                  &tickets.lock);
            
            if (tickets.total[location] > 0)
                res = reserve_for(location, sequence_number, price);
            else
                res = 0;
        }
    else
    res = 0;
    }
    pthread_mutex_unlock(&tickets.lock);
 
    return res;
 }
 
void finalize_ticket(int location) //completes ticket purchase reduces the amount of avaiable tix
{
    pthread_mutex_lock(&tickets.lock);
    tickets.reserved[location]--;
    pthread_cond_broadcast(&tickets.CV[location]);
    pthread_mutex_unlock(&tickets.lock);
}
 
void release_reserved_ticket(int location)//releases the reserved ticket and increases the amount of available tickets
{              
    pthread_mutex_lock(&tickets.lock);
    tickets.reserved[location]--;
    tickets.total[location]++;
    pthread_cond_broadcast(&tickets.CV[location]);
    pthread_mutex_unlock(&tickets.lock);
}
void get_quote(struct thread_info* ti, int location) //get_quote determines the price of the requested ticket for a specified location
{
    int sequence_number = 0;
    int price = 0;
    int res = reserve_ticket(location, &sequence_number, &price);
 
    if (res)
    {
        write_msg(ti, price);
        int msg = 0;
        if (read_msg(ti, &msg) == 0)
            return;
 
        if (msg == I_ACCEPT)
        {
            write_msg(ti, sequence_number);
            finalize_ticket(location);
        }
        else
        {
            write_msg(ti, REJECT_ACKNOWLEDGED);
            release_reserved_ticket(location);
        }
    }
    else
        write_msg(ti, TICKET_SOLD_OUT);
}


