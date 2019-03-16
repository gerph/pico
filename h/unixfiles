/*******************************************************************
 * File:     unixfiles.h
 * Purpose:  A unixlike files library for RISC OS - specifically for libwad
 * Author:   Justin Fletcher
 ******************************************************************/

#ifndef UNIXFILES_H
#define UNIXFILES_H

#include <stdlib.h>

#define O_RDONLY (1)
#define O_WRONLY (2)
#define O_RDWR   (3)
#define O_BINARY (4)
#define O_TRUNC  (8)
#define O_CREAT  (16)

#define SEEK_SET 0 /* start of stream (see fseek) */
#define SEEK_CUR 1 /* current position in stream (see fseek) */
#define SEEK_END 2 /* end of stream (see fseek) */

/* Open a file */
int open(char *file,int type,...);

/* Read some data */
int read(int fd,void *buf, size_t length);

/* Write some data */
int write(int fd,const void *buf, size_t length);

/* Close a file */
int close(int fd);

/* How long is this file */
int filelength (int fd);

/* Where are we in it ? */
long tell (int fd);

/* goto? */
long lseek (int fd,long pos, int from);

/* Copy a string */
char *strdup(const char *);

/* Nasty, yucky hack to make fileno work */
#define fileno(x) (x->__file)

/* Set the type of a file */
/* Not very unixlike, but incredibly useful */
int settype(char *file, int type);

/*************************************************** Gerph *********
 Function:     getcwd
 Description:  Return the current working directory
 Parameters:   buffer-> buffer to write to
               bufferlen = length of buffer to write to
 Returns:      pointer to buffer, or NULL if failed
 ******************************************************************/
char *getcwd(char *buffer,size_t bufferlen);

#endif
