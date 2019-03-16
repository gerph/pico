/*******************************************************************
 * File:     unixfiles.h
 * Purpose:  A unixlike files library for RISC OS - specifically for libwad
 * Author:   Justin Fletcher
 * Date:     11 Oct 1998
 * Note:     Very hastily thrown together
 ******************************************************************/

#include <stdlib.h>
#include <string.h>
#include <stdio.h>
#include "swis.h"
#include "kernel.h"
#include "unixfiles.h"

/* Identifier */
/* #include "VersionNum" */
/* static const char unixfiles[]="UnixFiles "Module_FullVersionAndDate; */

/* #define DEBUG */


static int openfiles[255];
static int openfilesc=0;


/* Close all open files on exit */
static void closeall(void)
{
  int last=openfilesc;
  while (last!=openfilesc && openfilesc>0)
  {
    close(openfiles[0]);
  }
}

/* Open a file */
int open(char *file,int type,...)
{
  int opentype=0;
  _kernel_swi_regs ARM;
  if (type & O_WRONLY)
  {
    if (type & O_CREAT)
      opentype=0x80;
    else
      opentype=0xC0;
  }
  else
    opentype=0x40;

  ARM.r[0]=opentype;
  ARM.r[1]=(int)file;
  if (_kernel_swi(OS_Find,&ARM,&ARM))
    return -1;
  if (ARM.r[0]==0)
    return -1;

  {
    static int registered=0;
    if (!registered)
    {
      atexit(closeall);
      registered=1;
    }
    openfiles[openfilesc++]=ARM.r[0];
  }

  return ARM.r[0];
}

/* Read some data */
int read(int fd,void *buf, size_t length)
{
  _kernel_swi_regs ARM;

  if (fd<=0)
    return -1; /*FAILED!*/
  ARM.r[0]=4;
  ARM.r[1]=fd;
  ARM.r[2]=(int)buf;
  ARM.r[3]=length;
#ifdef DEBUG
  printf("Read @ &%x, of &%x\n",tell(fd),length);
#endif
  if (_kernel_swi(OS_GBPB,&ARM,&ARM))
    return -1;
  return length-ARM.r[3];
}

/* Write some data */
int write(int fd,const void *buf, size_t length)
{
  _kernel_swi_regs ARM;

  if (fd<=0)
    return -1; /*FAILED!*/
  ARM.r[0]=2;
  ARM.r[1]=fd;
  ARM.r[2]=(int)buf;
  ARM.r[3]=length;
  if (_kernel_swi(OS_GBPB,&ARM,&ARM))
    return -1;
  return length-ARM.r[3];
}

/* Close a file */
int close(int fd)
{
  _kernel_swi_regs ARM;

  if (fd<=0)
    return -1; /*FAILED!*/
  ARM.r[0]=0;
  ARM.r[1]=fd;
  {
    int i,o;
    o=0;
    for (i=0; i<openfilesc;i++)
    {
      if (openfiles[i]!=fd)
        openfiles[o++]=openfiles[i];
    }
    openfilesc=o;
  }
  if (_kernel_swi(OS_Find,&ARM,&ARM))
    return -1;
  return 0;
}

/* How long is this file */
int filelength (int fd)
{
  _kernel_swi_regs ARM;

  if (fd<=0)
    return -1; /*FAILED!*/
  ARM.r[0]=2;
  ARM.r[1]=fd;
  if (_kernel_swi(OS_Args,&ARM,&ARM))
    return -1;
  return ARM.r[2];
}

/* Where are we in it ? */
long tell (int fd)
{
  _kernel_swi_regs ARM;

  if (fd<=0)
    return -1; /*FAILED!*/
  ARM.r[0]=0;
  ARM.r[1]=fd;
  if (_kernel_swi(OS_Args,&ARM,&ARM))
    return -1;
  return ARM.r[2];
}

/* Goto ? */
long lseek (int fd,long pos, int from)
{
  _kernel_swi_regs ARM;

  if (fd<=0)
    return -1; /*FAILED!*/
  if (from==SEEK_END)
    pos+=filelength(fd);
  else if (from==SEEK_CUR)
    pos+=tell(fd);
  else if (from!=SEEK_SET)
    return -1; /*You used a stupid type */
  ARM.r[0]=1;
  ARM.r[1]=fd;
  ARM.r[2]=pos;
#ifdef DEBUG
  printf("Seek &%x\n",pos);
#endif
  if (_kernel_swi(OS_Args,&ARM,&ARM))
    return -1;
  return ARM.r[2];
}

/* Duplicate a string */
char *strdup(const char *str)
{
  char *newstr=malloc(strlen(str)+1);
  if (newstr==NULL) return NULL;
  else return strcpy(newstr,str);
}

/* Set the type of a file */
int settype(char *file, int type)
{
  _kernel_swi_regs ARM;
  ARM.r[0]=18;
  ARM.r[1]=(int)file;
  ARM.r[2]=type;
  if (_kernel_swi(OS_File,&ARM,&ARM))
    return -1;
  return 0;
}

/*************************************************** Gerph *********
 Function:     getcwd
 Description:  Return the current working directory
 Parameters:   buffer-> buffer to write to
               bufferlen = length of buffer to write to
 Returns:      pointer to buffer, or NULL if failed
 ******************************************************************/
char *getcwd(char *buffer,size_t bufferlen)
{
  if (_swix(OS_FSControl,_INR(0,5),
                                   37,
                                   "@",
                                   buffer,
                                   0,
                                   0,
                                   bufferlen))
    return NULL;
  return buffer;
}
