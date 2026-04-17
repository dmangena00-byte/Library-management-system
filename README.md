#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <time.h>

#define MAX_BOOKS   500
#define MAX_MEMBERS 200
#define FINE_PER_DAY 2.0
#define MAX_DAYS     14

/* ─── Data Structures ─────────────────────────────────────────── */

typedef struct {
    int   bookID;
    char  title[100];
    char  author[60];
    char  genre[40];
    int   quantity;
    int   available;
    float price;
} Book;

typedef struct {
    int   memberID;
    char  name[60];
    char  email[80];
    char  phone[15];
    int   booksIssued;
    float totalFine;
} Member;

typedef struct {
    int  transID;
    int  memberID;
    int  bookID;
    char issueDate[12];
    char dueDate[12];
    char returnDate[12];
    int  returned;
    float fine;
} Transaction;

/* ─── File Names ──────────────────────────────────────────────── */
#define BOOKS_FILE  "books.dat"
#define MEMBERS_FILE "members.dat"
#define TRANS_FILE  "transactions.dat"

/* ─── Utility: get today's date string ───────────────────────── */
void getDate(char *buf) {
    time_t t = time(NULL);
    struct tm *tm = localtime(&t);
    sprintf(buf, "%02d/%02d/%04d",
            tm->tm_mday, tm->tm_mon + 1, tm->tm_year + 1900);
}

/* parse dd/mm/yyyy → days since epoch (approx) */
long dateToDays(const char *d) {
    int day, mon, yr;
    sscanf(d, "%d/%d/%d", &day, &mon, &yr);
    return yr * 365L + mon * 30 + day;
}

/* ─── Book Management ─────────────────────────────────────────── */

void addBook() {
    Book b;
    printf("\n  Enter Book ID      : "); scanf("%d",  &b.bookID);
    printf("  Enter Title        : "); scanf(" %[^\n]", b.title);
    printf("  Enter Author       : "); scanf(" %[^\n]", b.author);
    printf("  Enter Genre        : "); scanf(" %[^\n]", b.genre);
    printf("  Enter Quantity     : "); scanf("%d",  &b.quantity);
    printf("  Enter Price (Rs.)  : "); scanf("%f",  &b.price);
    b.available = b.quantity;

    FILE *fp = fopen(BOOKS_FILE, "ab");
    if (!fp) { printf("  [ERROR] Cannot open books file.\n"); return; }
    fwrite(&b, sizeof(Book), 1, fp);
    fclose(fp);
    printf("  [OK] Book added successfully (ID: %d).\n", b.bookID);
}

void displayAllBooks() {
    FILE *fp = fopen(BOOKS_FILE, "rb");
    if (!fp) { printf("  No book records found.\n"); return; }
    Book b;
    printf("\n  %-6s %-35s %-22s %-5s %-5s\n",
           "ID", "Title", "Author", "Qty", "Avail");
    printf("  %s\n", "-----------------------------------------------------------------------");
    while (fread(&b, sizeof(Book), 1, fp))
        printf("  %-6d %-35s %-22s %-5d %-5d\n",
               b.bookID, b.title, b.author, b.quantity, b.available);
    fclose(fp);
}

void searchBook() {
    int id; char kw[60];
    printf("\n  1. Search by ID   2. Search by Title\n  Choice: ");
    int ch; scanf("%d", &ch);
    FILE *fp = fopen(BOOKS_FILE, "rb");
    if (!fp) { printf("  No records.\n"); return; }
    Book b; int found = 0;
    if (ch == 1) {
        printf("  Enter Book ID: "); scanf("%d", &id);
        while (fread(&b, sizeof(Book), 1, fp))
            if (b.bookID == id) {
                printf("\n  ID: %d | %s by %s | Available: %d/%d\n",
                       b.bookID, b.title, b.author, b.available, b.quantity);
                found = 1; break;
            }
    } else {
        printf("  Enter keyword: "); scanf(" %[^\n]", kw);
        while (fread(&b, sizeof(Book), 1, fp))
            if (strstr(b.title, kw)) {
                printf("  ID: %d | %s by %s | Available: %d\n",
                       b.bookID, b.title, b.author, b.available);
                found = 1;
            }
    }
    if (!found) printf("  Book not found.\n");
    fclose(fp);
}

void deleteBook() {
    int id; printf("\n  Enter Book ID to delete: "); scanf("%d", &id);
    FILE *fp = fopen(BOOKS_FILE, "rb");
    FILE *tmp = fopen("tmp_books.dat", "wb");
    if (!fp || !tmp) { printf("  [ERROR] File error.\n"); return; }
    Book b; int found = 0;
    while (fread(&b, sizeof(Book), 1, fp))
        if (b.bookID != id) fwrite(&b, sizeof(Book), 1, tmp);
        else found = 1;
    fclose(fp); fclose(tmp);
    remove(BOOKS_FILE);
    rename("tmp_books.dat", BOOKS_FILE);
    printf(found ? "  [OK] Book deleted.\n" : "  Book ID not found.\n");
}

/* ─── Member Management ───────────────────────────────────────── */

void addMember() {
    Member m;
    printf("\n  Enter Member ID    : "); scanf("%d",  &m.memberID);
    printf("  Enter Name         : "); scanf(" %[^\n]", m.name);
    printf("  Enter Email        : "); scanf(" %[^\n]", m.email);
    printf("  Enter Phone        : "); scanf(" %[^\n]", m.phone);
    m.booksIssued = 0; m.totalFine = 0;

    FILE *fp = fopen(MEMBERS_FILE, "ab");
    if (!fp) { printf("  [ERROR] Cannot open members file.\n"); return; }
    fwrite(&m, sizeof(Member), 1, fp);
    fclose(fp);
    printf("  [OK] Member registered (ID: %d).\n", m.memberID);
}

void displayMembers() {
    FILE *fp = fopen(MEMBERS_FILE, "rb");
    if (!fp) { printf("  No member records found.\n"); return; }
    Member m;
    printf("\n  %-6s %-25s %-28s %-13s %-5s\n",
           "ID", "Name", "Email", "Phone", "Books");
    printf("  %s\n", "------------------------------------------------------------------------");
    while (fread(&m, sizeof(Member), 1, fp))
        printf("  %-6d %-25s %-28s %-13s %-5d\n",
               m.memberID, m.name, m.email, m.phone, m.booksIssued);
    fclose(fp);
}

/* ─── Issue / Return ──────────────────────────────────────────── */

void issueBook() {
    int mid, bid;
    printf("\n  Enter Member ID : "); scanf("%d", &mid);
    printf("  Enter Book ID   : "); scanf("%d", &bid);

    /* update book availability */
    FILE *fp = fopen(BOOKS_FILE, "r+b");
    if (!fp) { printf("  [ERROR] Books file missing.\n"); return; }
    Book b; int found = 0; long pos;
    while ((pos = ftell(fp)), fread(&b, sizeof(Book), 1, fp))
        if (b.bookID == bid) {
            if (b.available <= 0) { printf("  No copies available.\n"); fclose(fp); return; }
            b.available--;
            fseek(fp, pos, SEEK_SET);
            fwrite(&b, sizeof(Book), 1, fp);
            found = 1; break;
        }
    fclose(fp);
    if (!found) { printf("  Book not found.\n"); return; }

    /* log transaction */
    Transaction t;
    t.transID = (int)time(NULL);
    t.memberID = mid; t.bookID = bid; t.returned = 0; t.fine = 0;
    getDate(t.issueDate);
    long due = dateToDays(t.issueDate) + MAX_DAYS;
    sprintf(t.dueDate, "in %d days", MAX_DAYS);
    strcpy(t.returnDate, "N/A");

    FILE *tf = fopen(TRANS_FILE, "ab");
    fwrite(&t, sizeof(Transaction), 1, tf);
    fclose(tf);
    printf("  [OK] Book issued. Due in %d days. Fine: Rs.%.0f/day if late.\n",
           MAX_DAYS, FINE_PER_DAY);
}

void returnBook() {
    int mid, bid;
    printf("\n  Enter Member ID : "); scanf("%d", &mid);
    printf("  Enter Book ID   : "); scanf("%d", &bid);
    printf("  Days since issue: ");
    int days; scanf("%d", &days);

    float fine = 0;
    if (days > MAX_DAYS) {
        fine = (days - MAX_DAYS) * FINE_PER_DAY;
        printf("  Overdue by %d day(s). Fine = Rs. %.2f\n", days - MAX_DAYS, fine);
    } else {
        printf("  Returned on time. No fine.\n");
    }

    /* restore availability */
    FILE *fp = fopen(BOOKS_FILE, "r+b");
    Book b; long pos;
    while ((pos = ftell(fp)), fread(&b, sizeof(Book), 1, fp))
        if (b.bookID == bid) {
            b.available++;
            fseek(fp, pos, SEEK_SET);
            fwrite(&b, sizeof(Book), 1, fp);
            break;
        }
    fclose(fp);
    printf("  [OK] Book returned successfully.\n");
}

/* ─── Report ──────────────────────────────────────────────────── */

void generateReport() {
    int books = 0, members = 0, issued = 0;
    FILE *fp;
    Book b; Member m; Transaction t;

    fp = fopen(BOOKS_FILE, "rb");
    if (fp) { while (fread(&b, sizeof(Book), 1, fp)) { books++; issued += (b.quantity - b.available); } fclose(fp); }

    fp = fopen(MEMBERS_FILE, "rb");
    if (fp) { while (fread(&m, sizeof(Member), 1, fp)) members++; fclose(fp); }

    printf("\n  ===== LIBRARY REPORT =====\n");
    printf("  Total Books      : %d\n", books);
    printf("  Total Members    : %d\n", members);
    printf("  Books Issued     : %d\n", issued);
    printf("  ==========================\n");
}

/* ─── Main Menu ───────────────────────────────────────────────── */

int main() {
    int choice, sub;
    printf("\n  ============================================\n");
    printf("     CHANDIGARH UNIVERSITY LIBRARY SYSTEM    \n");
    printf("  ============================================\n");

    do {
        printf("\n  1. Book Management\n");
        printf("  2. Member Management\n");
        printf("  3. Issue / Return Book\n");
        printf("  4. Generate Report\n");
        printf("  0. Exit\n");
        printf("  Enter choice: "); scanf("%d", &choice);

        switch (choice) {
            case 1:
                printf("\n  1.Add  2.Display  3.Search  4.Delete\n  Choice: ");
                scanf("%d", &sub);
                if      (sub == 1) addBook();
                else if (sub == 2) displayAllBooks();
                else if (sub == 3) searchBook();
                else if (sub == 4) deleteBook();
                break;
            case 2:
                printf("\n  1.Register  2.View All\n  Choice: ");
                scanf("%d", &sub);
                if      (sub == 1) addMember();
                else if (sub == 2) displayMembers();
                break;
            case 3:
                printf("\n  1.Issue  2.Return\n  Choice: ");
                scanf("%d", &sub);
                if      (sub == 1) issueBook();
                else if (sub == 2) returnBook();
                break;
            case 4:
                generateReport();
                break;
            case 0:
                printf("\n  Goodbye!\n");
                break;
            default:
                printf("  Invalid choice.\n");
        }
    } while (choice != 0);

    return 0;
}
	◦
