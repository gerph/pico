/*
 * Directory manipulation a-la sys/dir.h
 *
 * Written by Justin Fletcher (2000-2001).
 */

#include <stdlib.h>
#include <stdio.h>
#include <string.h>
#include "swis.h"
#include "unixdirs.h"

/* Define this to debug this file */
/* #define DEBUG */

#ifdef DEBUG
#define dprintf if (1) printf
#else
#define dprintf if (0) printf
#endif

/* The blocks dealt with by dir_getentry */
typedef struct direntry_s {
  int  load;
  int  exec;
  int  length;
  int  attribs;
  int  objtype;
  int  filetype;
  char name[1];
} direntry;

/*********************************************** <c> Gerph *********
 Function:     dir_getentry
 Description:  Read an entry from a directory
 Parameters:   dir-> directory name
               match-> what to match, or NULL for all
               count-> counter to update for directory, NULL initially
 Returns:      directory entry, or NULL if none
 ******************************************************************/
static direntry *dir_getentry(const char *dir,const char *match, int *count)
{
  int buffer[MAXNAMLEN/sizeof(int)];
  int read=0;
  int namelen;
  direntry *data;

  if (match==NULL) match="*";

  while (read==0)
  {
    _kernel_oserror *err;
    err=_swix(OS_GBPB,_INR(0,6)|_OUTR(3,4),
                      12,dir,buffer,1,*count,sizeof(buffer),
                      match,&read,count);
    if (err)
      return NULL;

    if (*count==-1)
      return NULL; /* all done */
  }
  namelen=strlen((char *)&buffer[6]); /* find name length */
  data=malloc(sizeof(direntry)+namelen+1);
  if (data==NULL)
    return NULL;
  memcpy(data,buffer,sizeof(direntry)+namelen+1); /* copy the data */
  return data; /* return the data */
}

/*********************************************** <c> Gerph *********
 Function:     dir_disposeentry
 Description:  Remove information about a directory entry we've read
 Parameters:   ent-> the entry
 Returns:      none
 ******************************************************************/
static void dir_disposeentry(direntry *ent)
{
  if (ent!=NULL)
    free(ent);
}

DIR *opendir(const char * path)
{
  DIR *newdir;
  char *dirblock=NULL;
  int dirblocklen=0;
  int count = 0;
  direntry *ent;
  dprintf("OpenDir: %s\n",path);
  while ((ent=dir_getentry(path,NULL,&count))!=NULL)
  {
    char *block=realloc(dirblock,dirblocklen+sizeof(struct dirent));
    struct dirent *dent=(struct dirent *)&block[dirblocklen];
    if (block==NULL)
    {
      free(block);
      return NULL;
    }
    dirblocklen+=sizeof(struct dirent);
    dirblock=block;

    dent->d_fileno = 0; /* NOT IMPLEMENTED!!!! */
    dent->d_reclen = sizeof(struct dirent); /* Not efficient */
    dent->d_type = (ent->objtype & 2) ? DT_DIR :
                   (ent->objtype & 1) ? DT_REG : 0;
    dent->d_namlen = strlen(ent->name);
    strcpy(dent->d_name,ent->name);

    dprintf("Added file %s\n",ent->name);

    dir_disposeentry(ent);
  }

  newdir=malloc(sizeof(DIR));
  if (newdir==NULL)
  {
    free(dirblock);
    return NULL;
  }
  newdir->dd_fd = 0; /* NOT IMPLEMENTED!!!! */
  newdir->dd_loc = 0; /* Start of processing */
  newdir->dd_size = 0; /* NOT IMPLEMENTED!!!! */
  newdir->dd_buf = dirblock;
  newdir->dd_len = dirblocklen;
  newdir->dd_seek = 0; /* NOT IMPLEMENTED!!!! */
  newdir->dd_rewind = 0; /* NOT IMPLEMENTED!!!! */

  dprintf("Returning directory point %p\n",newdir);
  return newdir;
}

int closedir(DIR *dir)
{
  if (dir)
  {
    free(dir);
    free(dir->dd_buf);
  }
  return 0;
}

struct dirent * readdir(DIR *dir)
{
  long loc=dir->dd_loc;
  struct direct *dent=(struct direct *)&dir->dd_buf[loc];
  if (dir==NULL)
    return NULL;

  dprintf("readdir(%p): loc = &%lx/&%x\n",dir,loc,dir->dd_len);

  if (loc == dir->dd_len)
    return NULL; /* Finished reading */

  dir->dd_loc=loc+dent->d_reclen;

  return dent;
}

static struct timespec ro5bytetotimespec(unsigned long b[2])
{
  struct timespec ts;
  unsigned long t1, t2, tc;

  t1 = (b[0]);
  t2 = (b[1] & 0xff);

  tc = 0x6e996a00U;
  if (t1 < tc)
    t2--;
  t1 -= tc;
  t2 -= 0x33;			/* 00:00:00 Jan. 1 1970 = 0x336e996a00 */

  t1 = (t1 / 100) + (t2 * 42949673U);	/* 0x100000000 / 100 = 42949672.96 */
  t1 -= (t2 / 25);		/* compensate for .04 error */

  ts.ts_sec=t1;
  ts.ts_nsec=0;  /* Inaccurate, but what the hell */

  return ts;
}

int stat(const char *file,struct stat *st)
{
  unsigned long addrs[2];
  unsigned long objtype;
  unsigned long length;
  unsigned long attrs;

  dprintf("Stat file: %s\n",file);

  if (_swix(OS_File,_INR(0,1)|_OUT(0)|_OUTR(2,5),
                                        5,
                                        file,

                                        &objtype,
                                        &addrs[1],
                                        &addrs[0],
                                        &length,
                                        &attrs))
    return 1;

  st->st_dev=0;
  st->st_ino=0; /* Inode not supported - think about this */
  st->st_mode=0;
  if (attrs & (1<<0)) st->st_mode|=S_IRUSR;
  if (attrs & (1<<1)) st->st_mode|=S_IWUSR;
  if (attrs & (1<<4)) st->st_mode|=S_IRGRP | S_IROTH;
  if (attrs & (1<<5)) st->st_mode|=S_IWGRP | S_IWOTH;
  if ((attrs & (1<<0)) && (objtype & 2)) st->st_mode|=S_IXUSR;
  if ((attrs & (1<<4)) && (objtype & 2)) st->st_mode|=S_IXGRP | S_IXOTH;

  /* Directories cannot be regular files according to the headers I have */
  if (objtype & 2) st->st_mode|=S_IFDIR;
  else if (objtype & 1) st->st_mode|=S_IFREG;

  st->st_nlink=1; /* Hard links not supported */

  st->st_uid=0;
  st->st_gid=0;
  st->st_rdev=0;
  st->st_atimespec=
  st->st_mtimespec=
  st->st_ctimespec=ro5bytetotimespec(addrs);
  st->st_size=length;
  st->st_blksize=512; /* For general compatibility */
  st->st_blocks=(length+(st->st_blksize-1)) / st->st_blksize;
  st->st_flags=0;
  if ((addrs[1] & 0xfff00000) == 0xfff00000)
    st->st_gen = (addrs[1] & 0xfff00) >> 8;
  else
    st->st_gen=-1;
  return 0;
}

int lstat(const char *file,struct stat *st)
{
  return stat(file,st);
}
