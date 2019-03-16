/*
 * Directory manipulation a-la sys/dir.h
 *
 * Written by Justin Fletcher (2000-2001).
 */

#ifndef UNIX_DIRS_H
#define UNIX_DIRS_H

/* For backwards compatibility with BSD.  */
#define d_ino	d_fileno


/*
 * File types
 */
#define	DT_UNKNOWN	 0
#define	DT_FIFO		 1
#define	DT_CHR		 2
#define	DT_DIR		 4
#define	DT_BLK		 6
#define	DT_REG		 8
#define	DT_LNK		10
#define	DT_SOCK		12


/* definitions for library routines operating on directories. */
#define	DIRBLKSIZ	1024

/* structure describing an open directory. */
typedef struct _dirdesc {
	int	dd_fd;		/* file descriptor associated with directory */
	long	dd_loc;		/* offset in current buffer */
	long	dd_size;	/* amount of data returned by getdirentries */
	char	*dd_buf;	/* data buffer */
	int	dd_len;		/* size of data buffer */
	long	dd_seek;	/* magic cookie returned by getdirentries */
	long	dd_rewind;	/* magic cookie for rewinding */
} DIR;

/*
 * The dirent structure defines the format of directory entries returned by
 * the getdirentries(2) system call.
 *
 * A directory entry has a struct dirent at the front of it, containing its
 * inode number, the length of the entry, and the length of the name
 * contained in the entry.  These are followed by the name padded to a 4
 * byte boundary with null bytes.  All names are guaranteed null terminated.
 * The maximum length of a name in a directory is MAXNAMLEN.
 */

struct dirent {
	unsigned long	d_fileno;	/* file number of entry */
	unsigned short	d_reclen;	/* length of this record */
	unsigned char	d_type; 	/* file type, see below */
	unsigned char	d_namlen;	/* length of string in d_name */
#define	MAXNAMLEN	255
	char	d_name[MAXNAMLEN + 1];	/* name must be no longer than this */
};

#define direct dirent


/* Open a directory stream on name. Return a dir stream on
   the directory, or NULL if it could not be opened.  */
extern DIR *opendir (const char *__name);

/* Read a directory entry from dirp.
   Return a pointer to a struct dirent describing the entry,
   or NULL for EOF or error.  The storage returned may be overwritten
   by a later readdir call on the same DIR stream, or by closedir.  */
extern struct dirent *readdir (DIR *__dirp);

/* Close the directory stream dirp. Return 0 if successful,
   -1 if not.  */
extern int closedir (DIR *__dirp);

/******************************** STAT **************************************/

/*
 * Structure defined by POSIX.4 to be like a timeval.
 */
struct timespec {
	long	ts_sec;		/* seconds */
	long	ts_nsec;	/* and nanoseconds */
};


typedef unsigned long ino_t;
typedef unsigned long mode_t;
typedef unsigned long nlink_t;

struct stat {
        unsigned short  st_dev;         /* inode's device */
        ino_t   st_ino;                 /* inode's number */
        mode_t  st_mode;                /* inode protection mode */
        nlink_t st_nlink;               /* number of hard links */
        unsigned short  st_uid;         /* user ID of the file's owner */
        unsigned short  st_gid;         /* group ID of the file's group */
        unsigned short  st_rdev;        /* device type */
        long    st_size;                /* file size, in bytes */
        struct  timespec st_atimespec;  /* time of last access */
        struct  timespec st_mtimespec;  /* time of last data modification */
        struct  timespec st_ctimespec;  /* time of last file status change */
        long    st_blksize;             /* optimal blocksize for I/O */
        long    st_blocks;              /* blocks allocated for file */
        unsigned long   st_flags;       /* user defined flags for file */
        unsigned long   st_gen;         /* file generation number */
};
#define st_atime st_atimespec.ts_sec
#define st_mtime st_mtimespec.ts_sec
#define st_ctime st_ctimespec.ts_sec

#define S_ISUID 0004000                 /* set user id on execution */
#define S_ISGID 0002000                 /* set group id on execution */
#ifndef _POSIX_SOURCE
#define S_ISTXT 0001000                 /* sticky bit */
#endif

#define S_IRWXU 0000700                 /* RWX mask for owner */
#define S_IRUSR 0000400                 /* R for owner */
#define S_IWUSR 0000200                 /* W for owner */
#define S_IXUSR 0000100                 /* X for owner */

#ifndef _POSIX_SOURCE
#define S_IREAD         S_IRUSR
#define S_IWRITE        S_IWUSR
#define S_IEXEC         S_IXUSR
#endif

#define S_IRWXG 0000070                 /* RWX mask for group */
#define S_IRGRP 0000040                 /* R for group */
#define S_IWGRP 0000020                 /* W for group */
#define S_IXGRP 0000010                 /* X for group */

#define S_IRWXO 0000007                 /* RWX mask for other */
#define S_IROTH 0000004                 /* R for other */
#define S_IWOTH 0000002                 /* W for other */
#define S_IXOTH 0000001                 /* X for other */

#define S_IFMT   0170000                /* type of file mask */
#define S_IFIFO  0010000                /* named pipe (fifo) */
#define S_IFCHR  0020000                /* character special */
#define S_IFDIR  0040000                /* directory */
#define S_IFBLK  0060000                /* block special */
#define S_IFREG  0100000                /* regular */
#define S_IFLNK  0120000                /* symbolic link */
#define S_IFSOCK 0140000                /* socket */
#define S_ISVTX  0001000                /* save swapped text even after use */

int     stat(const char *, struct stat *);
int     lstat(const char *, struct stat *);

#endif
