#if	!defined(lint) && !defined(DOS)
static char rcsid[] = "$Id: os_arc.c,v 1.00 29 Dec 1996 18:31.13 Fletcher $";
#endif
/*
 * Program:	Operating system dependent routines - Acorn RO3.1 +
 *
 *
 * Michael Seibel
 * Networks and Distributed Computing
 * Computing and Communications
 * University of Washington
 * Administration Builiding, AG-44
 * Seattle, Washington, 98195, USA
 * Internet: mikes@cac.washington.edu
 *
 * Please address all bugs and comments to "pine-bugs@cac.washington.edu"
 *
 *
 * Pine and Pico are registered trademarks of the University of Washington.
 * No commercial use of these trademarks may be made without prior written
 * permission of the University of Washington.
 *
 * Pine, Pico, and Pilot software and its included text are Copyright
 * 1989-1996 by the University of Washington.
 *
 * The full text of our legal notices is contained in the file called
 * CPYRIGHT, included with this distribution.
 *
 *
 * Notes:
 *
 * - SGI IRIX 4.0.1 port by:
 *       johnb@edge.cis.mcmaster.ca,  2 April 1992
 *
 * - Dynix/PTX port by:
 *       Donn Cave, UCS/UW, 15 April 1992
 *
 * - 3B2, 3b1/7300, SCO ports by:
 *       rll@felton.felton.ca.us, 7 Feb. 1993
 *
 * - Altos System V (asv) port by:
 *	 Tim Rice <tim@trr.metro.net>    6 Mar 96
 *
 * - Acorn RISC OS port by:
 *       Justin Fletcher <justin.fletcher@ntlworld.com>, 29 Dec 1996-11 Mar 2002
 *       (VERY heavily thrown together for pico - NOT guarenteed
 *        to work with any of the other programs)
 *
 * - Probably have to break this up into separate os_type.c files since
 *   the #ifdef's are getting a bit cumbersome.
 *
 */

#include <stdio.h>
#include <errno.h>
#include <setjmp.h>
#include <time.h>
#include "swis.h"

#include "headers.h"



void sleep(int seconds)
{
  clock_t time=clock();
  /* printf("sleeping for %i seconds\n", seconds); */
  while (clock()-time < CLOCKS_PER_SEC*seconds)
  {
    /* Wait */
    _swix(OS_UpCall,_INR(0,1),6,0); /* Yield */
  }
}


/* This is a dim getpid implementation just to get out something that might be
   unique */
int getpid(void)
{
  static int pid=0;
  if (pid==0)
  {
    if (_swix(TaskWindow_TaskInfo,_IN(0)|_OUT(0),0,&pid))
      pid=0;
  }
  if (pid==0)
    _swix(OS_ReadMonotonicTime,_OUT(0),&pid);
  return pid;
}
