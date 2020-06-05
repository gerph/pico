#if	!defined(lint) && !defined(DOS)
static char rcsid[] = "$Id: header,v 1.2 1998/02/28 00:15:44 hubert Exp $";
#endif
/*
 * Program:	Routines to support file browser in pico and Pine composer
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
 * 1989-1998 by the University of Washington.
 *
 * The full text of our legal notices is contained in the file called
 * CPYRIGHT, included with this distribution.
 *
 */
#include "headers.h"

#ifdef __riscos
#include "swis.h"
#endif


int timeo = 0;
time_t time_of_last_input;
int (*pcollator)();

#if	defined(sv3) || defined(ct)
/*
 * Windowing structure to support JWINSIZE/TIOCSWINSZ/TIOCGWINSZ
 */
#define ENAMETOOLONG	78

struct winsize {
	unsigned short ws_row;       /* rows, in characters*/
	unsigned short ws_col;       /* columns, in character */
	unsigned short ws_xpixel;    /* horizontal size, pixels */
	unsigned short ws_ypixel;    /* vertical size, pixels */
};
#endif

#ifdef	SIGCHLD
static jmp_buf pico_child_state;
static short   pico_child_jmp_ok, pico_child_done;
#endif

#ifdef	MOUSE
static int mexist = 0;		/* is the mouse driver installed? */
static unsigned mnoop;
#endif

struct color_table {
    char *name;
    char *canonical_name;
    int   namelen;
    char *rgb;
    int   val;
};

static unsigned color_options;
#define	ANSI8_COLOR()	(color_options & COLOR_ANSI8_OPT)
#define	ANSI16_COLOR()	(color_options & COLOR_ANSI16_OPT)
#define	ANSI_COLOR()	(color_options & (COLOR_ANSI8_OPT | COLOR_ANSI16_OPT))
#define	END_PSEUDO_REVERSE	"EndInverse"
static COLOR_PAIR *the_rev_color, *the_normal_color;
static COLOR_PAIR *color_blasted_by_attrs;
static int pinvstate;	/* physical state of inverse (standout) attribute */
static int pboldstate;	/* physical state of bold attribute */
static int pulstate;	/* physical state of underline attribute */
static int rev_color_state;

static int boldstate;	/* should-be state of bold attribute */
static int ulstate;	/* should-be state of underline attribute */
static int invstate;	/* should-be state of Inverse, which could be a color
			   change or an actual setinverse */
#define A_UNKNOWN	-1


void	 kpinsert PROTO((char *, int, int));
void	 bail PROTO(());
void     flip_rev_color PROTO((int));
void     flip_bold PROTO((int));
void     flip_inv PROTO((int));
void     flip_ul PROTO((int));
void     reset_attr_state PROTO((void));
SigType	 do_hup_signal SIG_PROTO((int));
SigType	 rtfrmshell SIG_PROTO((int));
int	 pathcat PROTO((char *, char **, char *));
#ifdef	RESIZING
SigType	winch_handler SIG_PROTO((int));
#endif
#ifdef	SIGCHLD
SigType	child_handler SIG_PROTO((int));
#endif

extern char    *tgoto();
#if defined(USE_TERMINFO)
extern char    *tparm();
#endif
#ifndef __riscos
static void     putpad PROTO((char *));
#endif
static void	tinitcolor PROTO((void));
static int	tfgcolor PROTO((int));
static int	tbgcolor PROTO((int));
static struct color_table *init_color_table PROTO((void));
static void	free_color_table PROTO((struct color_table **));
static int      color_to_val PROTO((char *));

extern char   *_op, *_oc, *_setaf, *_setab, *_setf, *_setb, *_scp;
extern int	_colors;
static int     _color_inited, _using_color;
static char   *_nfcolor, *_nbcolor, *_rfcolor, *_rbcolor;
static char   *_last_fg_color, *_last_bg_color;
static int     _force_fg_color_change;
static int     _force_bg_color_change;

static struct color_table *color_tbl;
static int dummy;


/*
 * for alt_editor arg[] building
 */
#define	MAXARGS	10

#ifdef __riscos
int tt_oldcursorstate=0;
int tt_oldfunc8state=0;
int tt_oldfunc9state=0;
int tt_oldfuncCstate=0;

#define os_byte(a,x,y,reta) _swix(OS_Byte,_INR(0,2)|_OUT(0),a,x,y,reta)
#endif

/*
 * ttopen - this function is called once to set up the terminal device
 *          streams.  if called as pine composer, don't mess with
 *          tty modes, but set signal handlers.
 */
int ttopen(void)
{
#ifdef __riscos
  if(!Pmaster || (gmode ^ MDEXTFB))
    picosigs();
  _swix(OS_Byte,_INR(0,2)|_OUT(1),4,2,0,&tt_oldcursorstate);
  /* Change function key definitions */
  _swix(OS_Byte,_INR(0,2)|_OUT(1),225,0x80,0,&tt_oldfunc8state);
  _swix(OS_Byte,_INR(0,2)|_OUT(1),226,0x90,0,&tt_oldfunc9state);
  _swix(OS_Byte,_INR(0,2)|_OUT(1),221,0xC0,0,&tt_oldfuncCstate);
#else
    if(Pmaster == NULL){
	Raw(1);
#ifdef	MOUSE
	if(gmode & MDMOUSE)
	  init_mouse();
#endif	/* MOUSE */
	xonxoff_proc(preserve_start_stop);
    }

    picosigs();
#endif

    return(1);
}


/*
 * ttresize - recompute the screen dimensions if necessary, and then
 *	      adjust pico's internal buffers accordingly.
 */
void
ttresize ()
{
    int row = -1, col = -1;

    ttgetwinsz(&row, &col);
    resize_pico(row, col);
}


/*
 * picosigs - Install any handlers for the signals we're interested
 *	      in catching.
 */
void
picosigs()
{
#ifndef __riscos
    signal(SIGHUP,  do_hup_signal);	/* deal with SIGHUP */
#else
    signal(SIGINT, SIG_IGN);		/* Ignore Escape */
#endif
    signal(SIGTERM, do_hup_signal);	/* deal with SIGTERM */
#ifdef	SIGTSTP
    signal(SIGTSTP, SIG_DFL);
#endif
#ifdef	RESIZING
    signal(SIGWINCH, winch_handler); /* window size changes */
#endif
}


/*
 * ttclose - this function gets called just before we go back home to
 *           the command interpreter.  If called as pine composer, don't
 *           worry about modes, but set signals to default, pine will
 *           rewire things as needed.
 */
int ttclose(void)
{
#ifdef __riscos
    if(Pmaster){
	signal(SIGINT, SIG_DFL);
    }
    _swix(OS_Byte,_INR(0,2),4,tt_oldcursorstate,0);
    _swix(OS_Byte,_INR(0,2),225,tt_oldfunc8state,0);
    _swix(OS_Byte,_INR(0,2),226,tt_oldfunc9state,0);
    _swix(OS_Byte,_INR(0,2),221,tt_oldfuncCstate,0);
    /* Ensure that we end on a blank line */
    printf("\n");

#else

    if(Pmaster){
	signal(SIGHUP, SIG_DFL);
#ifdef  SIGCONT
	signal(SIGCONT, SIG_DFL);
#endif
#ifdef  RESIZING
	signal(SIGWINCH, SIG_DFL);
#endif
    }
    else{
	Raw(0);
#ifdef  MOUSE
	end_mouse();
#endif
    }
#endif

    return(1);
}


/*
 * ttgetwinsz - set global row and column values (if we can get them)
 *		and return.
 */
void
ttgetwinsz(row, col)
    int *row, *col;
{
#ifdef __riscos
  if (taskwindow())
  {
    if(*row < 0)
      *row = 24 - 1;
    if(*col <= 0)
      *col = 72;
  }
  else
  {
    if(*row < 0)
      *row = NROW - 1;
    if(*col <= 0)
      *col = NCOL;
  }
#else
    extern int _tlines, _tcolumns;

    if(*row < 0)
      *row = (_tlines > 0) ? _tlines - 1 : NROW - 1;
    if(*col <= 0)
      *col = (_tcolumns > 0) ? _tcolumns : NCOL;
#ifdef RESIZING
    {
	struct winsize win;

	if (ioctl(0, TIOCGWINSZ, &win) == 0) {	/* set to anything useful.. */
	    if(win.ws_row)			/* ... the tty drivers says */
	      *row = win.ws_row - 1;

	    if(win.ws_col)
	      *col = win.ws_col;
	}

	signal(SIGWINCH, winch_handler);	/* window size changes */
    }
#endif
#endif /* __riscos */

    if(*col > NLINE-1)
      *col = NLINE-1;
}


/*
 * ttputc - Write a character to the display.
 */
int ttputc(int c)
{
#ifdef __riscos
    if (_swix(OS_WriteC,_IN(0),c))
      return EOF;
    return c;
#else
    return(putc(c, stdout));
#endif
}


/*
 * ttflush - flush terminal buffer. Does real work where the terminal
 *           output is buffered up. A no-operation on systems where byte
 *           at a time terminal I/O is done.
 */
int ttflush()
{
#ifdef __riscos
    return 0;
#else
    return(fflush(stdout));
#endif
}


/*
 * ttgetc - Read a character from the terminal, performing no editing
 *          and doing no echo at all.
 *
 * Args: return_on_intr     -- Function to get a single character from stdin,
 *             recorder     -- If non-NULL, function used to record keystroke.
 *             bail_handler -- Function used to bail out on read error.
 *
 *  Returns: The character read from stdin.
 *           Return_on_intr is returned if read is interrupted.
 *           If read error, BAIL_OUT is returned unless bail_handler is
 *             non-NULL, in which case it is called (and usually it exits).
 *
 *      If recorder is non-null, it is used to record the keystroke.
 */
int
ttgetc(return_on_intr, recorder, bail_handler)
    int  return_on_intr;
    int  (*recorder) PROTO((int));
    void (*bail_handler)();
{
    int c;

    switch(c = read_one_char()){
      case READ_INTR:
	return(return_on_intr);

      case BAIL_OUT:
	if(bail_handler)
	  (*bail_handler)();
	else
	  return(BAIL_OUT);

      default:
        return(recorder ? (*recorder)(c) : c);
    }
}

/*
 * Simple version of ttgetc with simple error handling
 *
 *      Args:  recorder     -- If non-NULL, function used to record keystroke.
 *             bail_handler -- Function used to bail out on read error.
 *
 *  Returns: The character read from stdin.
 *           If read error, BAIL_OUT is returned unless bail_handler is
 *             non-NULL, in which case it is called (and usually it exits).
 *
 *      If recorder is non-null, it is used to record the keystroke.
 *      Retries if interrupted.
 */
int
simple_ttgetc(recorder, bail_handler)
    int  (*recorder) PROTO((int));
    void (*bail_handler)();
{
    int		  res;
    /* c was a char; now an int */
    unsigned int  c;

#ifdef __riscos
    c = read_one_char();
    /* printf("simple_ttgetc: %i (recorder=%p)\n", c, recorder); */
    if (c==BAIL_OUT)
      (*bail_handler)();
    else
      return(recorder ? (*recorder)((int)c) : (int)c);
    return 0; /* Just to keep compiler happy - otherwise we implicit return */
#else
    while((res = read(STDIN_FD, &c, 1)) <= 0)
      if(!(res < 0 && errno == EINTR))
	(*bail_handler)();

    return(recorder ? (*recorder)((int)c) : (int)c);
#endif
}

void
bail()
{
#ifdef __riscos
    printf("Bailing!\n");
    abort();
#else
    sleep(30);				/* see if os receives SIGHUP */
    kill(getpid(), SIGHUP);			/* eof or bad error */
#endif
}


#if	TYPEAH
/*
 * typahead - Check to see if any characters are already in the
 *	      keyboard buffer
 */
typahead()
{
#ifdef __riscos
    return 0;
#else
    int x;	/* holds # of pending chars */

    return((ioctl(0,FIONREAD,&x) < 0) ? 0 : x);
#endif
}
#endif


/*
 * ReadyForKey - return true if there's no timeout or we're told input
 *		 is available...
 */
ReadyForKey(timeout)
    int timeout;
{
    switch(input_ready(timeout)){
      case READY_TO_READ:
	return(1);
	break;

      case NO_OP_COMMAND:
      case NO_OP_IDLE:
      case READ_INTR:
	return(0);

      case BAIL_OUT:
      case PANIC_NOW:
	emlwrite("\007Problem reading from keyboard!", NULL);
#ifndef __riscos
	kill(getpid(), SIGHUP);	/* Bomb out (saving our work)! */
	/* no return */
#else
        printf("input_ready returned a fault!!\n");
        raise(SIGINT);
#endif
	/* no return */
    }

    /* can't happen */
    return(0);
}


/*
 * GetKey - Read in a key.
 * Do the standard keyboard preprocessing. Convert the keys to the internal
 * character set.  Resolves escape sequences and returns no-op if global
 * timeout value exceeded.
 */
GetKey()
{
    int    ch, status, cc;

    if(!ReadyForKey(FUDGE-5))
      return(NODATA);

    switch(status = kbseq(simple_ttgetc, NULL, bail, &ch)){
      case 0: 	/* regular character */
	break;

      case KEY_DOUBLE_ESC:
	/*
	 * Special hack to get around comm devices eating control characters.
	 */
	if(!ReadyForKey(5))
	  return(BADESC);		/* user typed ESC ESC, then stopped */
	else
	  ch = (*term.t_getchar)(NODATA, NULL, bail);

	ch &= 0x7f;
	if(isdigit((unsigned char)ch)){
	    int n = 0, i = ch - '0';

	    if(i < 0 || i > 2)
	      return(BADESC);		/* bogus literal char value */

	    while(n++ < 2){
		if(!ReadyForKey(5)
		   || (!isdigit((unsigned char) (ch =
				   (*term.t_getchar)(NODATA, NULL, bail)))
		       || (n == 1 && i == 2 && ch > '5')
		       || (n == 2 && i == 25 && ch > '5'))){
		    return(BADESC);
		}

		i = (i * 10) + (ch - '0');
	    }

	    ch = i;
	}
	else{
	    if(islower((unsigned char)ch))	/* canonicalize if alpha */
	      ch = toupper((unsigned char)ch);

	    return((isalpha((unsigned char)ch) || ch == '@'
		    || (ch >= '[' && ch <= '_'))
		    ? (CTRL | ch) : ((ch == ' ') ? (CTRL | '@') : ch));
	}

	break;

#ifdef MOUSE
      case KEY_XTERM_MOUSE:
	{
	    /*
	     * Special hack to get mouse events from an xterm.
	     * Get the details, then pass it past the keymenu event
	     * handler, and then to the installed handler if there
	     * is one...
	     */
	    static int down = 0;
	    int        x, y, button;
	    unsigned   cmd;

	    button = (*term.t_getchar)(NODATA, NULL, bail) & 0x03;

	    x = (*term.t_getchar)(NODATA, NULL, bail) - '!';
	    y = (*term.t_getchar)(NODATA, NULL, bail) - '!';

	    if(button == 0){
		down = 1;
		if(checkmouse(&cmd, 1, x, y))
		  return((int)cmd);
	    }
	    else if(down && button == 3){
		down = 0;
		if(checkmouse(&cmd, 0, x, y))
		  return((int)cmd);
	    }

	    return(NODATA);
	}

	break;
#endif /* MOUSE */

      case  KEY_UP		:
      case  KEY_DOWN		:
      case  KEY_RIGHT		:
      case  KEY_LEFT		:
      case  KEY_PGUP		:
      case  KEY_PGDN		:
      case  KEY_HOME		:
      case  KEY_END		:
      case  KEY_DEL		:
      case F1  :
      case F2  :
      case F3  :
      case F4  :
      case F5  :
      case F6  :
      case F7  :
      case F8  :
      case F9  :
      case F10 :
      case F11 :
      case F12 :
	return(status);

      case KEY_SWALLOW_Z:
	status = BADESC;
      case KEY_SWAL_UP:
      case KEY_SWAL_DOWN:
      case KEY_SWAL_LEFT:
      case KEY_SWAL_RIGHT:
	do
	  if(!ReadyForKey(2)){
	      status = BADESC;
	      break;
	  }
	while(!strchr("~qz", (*term.t_getchar)(NODATA, NULL, bail)));

	return((status == BADESC)
		? status
		: status - (KEY_SWAL_UP - KEY_UP));
	break;

      case KEY_KERMIT:
	do{
	    cc = ch;
	    if(!ReadyForKey(2))
	      return(BADESC);
	    else
	      ch = (*term.t_getchar)(NODATA, NULL, bail) & 0x7f;
	}while(cc != '\033' && ch != '\\');

	ch = NODATA;
	break;

      case BADESC:
	(*term.t_beep)();
	return(status);

      default:				/* punt the whole thing	*/
	(*term.t_beep)();
	break;
    }

    if(ch & 0x80 && Pmaster && Pmaster->hibit_entered)
      *Pmaster->hibit_entered = 1;

    if (ch >= 0x00 && ch <= 0x1F)       /* C0 control -> C-     */
      ch = CTRL | (ch+'@');

    return(ch);
}

#ifdef __riscos
#include "swis.h"
#endif

#define PATH_MAX 256

static int      tinforev PROTO((int));

/*
 * kbseq - looks at an escape sequence coming from the keyboard and
 *         compares it to a trie of known keyboard escape sequences, and
 *         returns the function bound to the escape sequence.
 *
 *     Args: getcfunc     -- Function to get a single character from stdin,
 *                           called with the next two arguments to this
 *                           function as its arguments.
 *           recorder     -- If non-NULL, function used to record keystroke.
 *           bail_handler -- Function used to bail out on read error.
 *           c            -- Pointer to returned character.
 *
 *  Returns: BADESC
 *           The escaped function.
 *           0 if a regular char with char stuffed in location c.
 */
kbseq(getcfunc, recorder, bail_handler, c)
    int  (*getcfunc)();
    int  (*recorder)();
    void (*bail_handler)();
    int   *c;
{
  /* JRF: b was a char; I've changed it to an int because a char can't
          store wide enough values. */
    register int      b;
    register int      first = 1;
    register KBESC_T *current;

    current = kbesc;
    if(current == NULL)				/* bag it */
      return(BADESC);

    while(1){
	b = *c = (*getcfunc)(recorder, bail_handler);

	/* printf("kbseq = %i\n", b); */

	while(current->value != b){
	    if(current->left == NULL){		/* NO MATCH */
		if(first)
		  return(0);			/* regular char */
		else
		  return(BADESC);
	    }
	    current = current->left;
	}

	if(current->down == NULL)		/* match!!!*/
	  return(current->func);
	else
	  current = current->down;

	first = 0;
    }
}


#define	newnode()	(KBESC_T *)malloc(sizeof(KBESC_T))
/*
 * kpinsert - insert a keystroke escape sequence into the global search
 *	      structure.
 */
void
kpinsert(kstr, kval, termcap_wins)
    char     *kstr;
    int       kval;
    int       termcap_wins;
{
    register	char	*buf;
    register	KBESC_T *temp;
    register	KBESC_T *trail;

    if(kstr == NULL)
      return;

    /*
     * Don't allow escape sequences that don't start with ESC unless
     * termcap_wins.  This is to protect against mistakes in termcap files.
     */
    if(!termcap_wins && *kstr != '\033')
      return;

    temp = trail = kbesc;
    buf = kstr;

    for(;;){
	if(temp == NULL){
	    temp = newnode();
	    temp->value = *buf;
	    temp->func = 0;
	    temp->left = NULL;
	    temp->down = NULL;
	    if(kbesc == NULL)
	      kbesc = temp;
	    else
	      trail->down = temp;
	}
	else{				/* first entry */
	    while((temp != NULL) && (temp->value != *buf)){
		trail = temp;
		temp = temp->left;
	    }

	    if(temp == NULL){   /* add new val */
		temp = newnode();
		temp->value = *buf;
		temp->func = 0;
		temp->left = NULL;
		temp->down = NULL;
		trail->left = temp;
	    }
	}

	if(*(++buf) == '\0')
	  break;
	else{
	    /*
	     * Ignore attempt to overwrite shorter existing escape sequence.
	     * That means that sequences with higher priority should be
	     * set up first, so if we want termcap sequences to override
	     * hardwired sequences, put the kpinsert calls for the
	     * termcap sequences first.  (That's what you get if you define
	     * TERMCAP_WINS.)
	     */
	    if(temp->func != 0)
	      return;

	    trail = temp;
	    temp = temp->down;
	}
    }

    /*
     * Ignore attempt to overwrite longer sequences we are a prefix
     * of (down != NULL) and exact same sequence (func != 0).
     */
    if(temp != NULL && temp->down == NULL && temp->func == 0)
      temp->func = kval;
}



/*
 * kbdestroy() - kills the key pad function key search tree
 *		 and frees all lines associated with it
 *
 *               Should be called with arg kbesc, the top of the tree.
 */
void
kbdestroy(kb)
    KBESC_T *kb;
{
    if(kb){
	kbdestroy(kb->left);
	kbdestroy(kb->down);
	free((char *)kb);
	kb = NULL;
    }
}


/*
 *     Start or end bold mode
 *
 * Result: escape sequence to go into or out of reverse color is output
 *
 *     (This is only called when there is a reverse color.)
 *
 * Arg   state = ON   set bold
 *               OFF  set normal
 */
void
flip_rev_color(state)
    int state;
{
    if((rev_color_state = state) == TRUE)
      (void)pico_set_colorp(the_rev_color, PSC_NONE);
    else
      pico_set_normal_color();
}


/*
 *     Start or end bold mode
 *
 * Result: escape sequence to go into or out of bold is output
 *
 * Arg   state = ON   set bold
 *               OFF  set normal
 */
void
flip_bold(state)
    int state;
{
#ifndef __riscos
    extern char *_setbold, *_clearallattr;

    if((pboldstate = state) == TRUE){
	if(_setbold != NULL)
	  putpad(_setbold);
    }
    else{
	if(_clearallattr != NULL){
	    if(!color_blasted_by_attrs)
	      color_blasted_by_attrs = pico_get_cur_color();

	    _force_fg_color_change = _force_bg_color_change = 1;
	    putpad(_clearallattr);
	    pinvstate = state;
	    pulstate = state;
	    rev_color_state = state;
	}
    }
#else
#endif
}


/*
 *     Start or end inverse mode
 *
 * Result: escape sequence to go into or out of inverse is output
 *
 * Arg   state = ON   set inverse
 *               OFF  set normal
 */
void
flip_inv(state)
    int state;
{
#ifndef __riscos
    extern char *_setinverse, *_clearinverse;

    if((pinvstate = state) == TRUE){
	if(_setinverse != NULL)
	  putpad(_setinverse);
    }
    else{
	/*
	 * Unfortunately, some termcap entries configure end standout to
	 * be clear all attributes.
	 */
	if(_clearinverse != NULL){
	    if(!color_blasted_by_attrs)
	      color_blasted_by_attrs = pico_get_cur_color();

	    _force_fg_color_change = _force_bg_color_change = 1;
	    putpad(_clearinverse);
	    pboldstate = (pboldstate == FALSE) ?  pboldstate : A_UNKNOWN;
	    pulstate = (pulstate == FALSE) ?  pulstate : A_UNKNOWN;
	    rev_color_state = A_UNKNOWN;
	}
    }
#endif
}


/*
 *     Start or end underline mode
 *
 * Result: escape sequence to go into or out of underline is output
 *
 * Arg   state = ON   set underline
 *               OFF  set normal
 */
void
flip_ul(state)
    int state;
{
#ifndef __riscos
    extern char *_setunderline, *_clearunderline;

    if((pulstate = state) == TRUE){
	if(_setunderline != NULL)
	  putpad(_setunderline);
    }
    else{
	/*
	 * Unfortunately, some termcap entries configure end underline to
	 * be clear all attributes.
	 */
	if(_clearunderline != NULL){
	    if(!color_blasted_by_attrs)
	      color_blasted_by_attrs = pico_get_cur_color();

	    _force_fg_color_change = _force_bg_color_change = 1;
	    putpad(_clearunderline);
	    pboldstate = (pboldstate == FALSE) ?  pboldstate : A_UNKNOWN;
	    pinvstate = (pinvstate == FALSE) ?  pinvstate : A_UNKNOWN;
	    rev_color_state = A_UNKNOWN;
	}
    }
#endif
}

void
StartInverse()
{
    invstate = TRUE;
    reset_attr_state();
}


void
EndInverse()
{
    invstate = FALSE;
    reset_attr_state();
}


int
InverseState()
{
    return(invstate);
}

int
StartBold()
{
    extern char *_setbold;

    boldstate = TRUE;
    reset_attr_state();
    return(_setbold != NULL);
}

void
EndBold()
{
    boldstate = FALSE;
    reset_attr_state();
}

void
StartUnderline()
{
    ulstate = TRUE;
    reset_attr_state();
}

void
EndUnderline()
{
    ulstate = FALSE;
    reset_attr_state();
}

void
reset_attr_state()
{
    /*
     * If we have to turn some attributes off, do that first since that
     * may turn off other attributes as a side effect.
     */
    if(boldstate == FALSE && pboldstate != boldstate)
      flip_bold(boldstate);

    if(ulstate == FALSE && pulstate != ulstate)
      flip_ul(ulstate);

    if(invstate == FALSE){
	if(pico_get_rev_color()){
	    if(rev_color_state != invstate)
	      flip_rev_color(invstate);
	}
	else{
	    if(pinvstate != invstate)
	      flip_inv(invstate);
	}
    }

    /*
     * Now turn everything on that needs turning on.
     */
    if(boldstate == TRUE && pboldstate != boldstate)
      flip_bold(boldstate);

    if(ulstate == TRUE && pulstate != ulstate)
      flip_ul(ulstate);

    if(invstate == TRUE){
	if(pico_get_rev_color()){
	    if(rev_color_state != invstate)
	      flip_rev_color(invstate);
	}
	else{
	    if(pinvstate != invstate)
	      flip_inv(invstate);
	}
    }

    if(color_blasted_by_attrs){
	(void)pico_set_colorp(color_blasted_by_attrs, PSC_NONE);
	free_color_pair(&color_blasted_by_attrs);
    }

}


static void
tinitcolor()
{
    if(_color_inited || panicking)
      return;

    if(ANSI_COLOR() || (_colors > 0 && ((_setaf && _setab) || (_setf && _setb)
/**** not sure how to do this yet
       || _scp
****/
	))){
	_color_inited = 1;
	color_tbl = init_color_table();

	if(ANSI_COLOR())
#ifdef __riscos
	{
          if (gmode&MDUSEANSI)
	    _swix(OS_Write0,_IN(0),"\033[39;49m");
	  else
	  {
	    /* Nothing to init */
	  }
	}
#else
	  putpad("\033[39;49m");
	else{
	    if(_op)
	      putpad(_op);
	    if(_oc)
	      putpad(_oc);
	}
#endif
    }
}


#if defined(USE_TERMCAP)
/*
 * Treading on thin ice here. Tgoto wasn't designed for this. It takes
 * arguments column and row. We only use one of them, so we put it in
 * the row argument. The 0 is just a placeholder.
 */
#define tparm(s, c) tgoto(s, 0, c)
#endif

static int
tfgcolor(color)
    int color;
{
    if(!_color_inited)
      tinitcolor();

    if(!_color_inited)
      return(-1);

    if((ANSI8_COLOR()  && (color < 0 || color >= 8)) ||
       (ANSI16_COLOR() && (color < 0 || color >= 16)) ||
       (!ANSI_COLOR()  && (color < 0 || color >= _colors)))
      return(-1);

    if(ANSI_COLOR()){
	char buf[10];

	if(color < 8)
	  sprintf(buf, "\033[3%cm", color + '0');
	else
	  sprintf(buf, "\033[9%cm", (color-8) + '0');

#ifdef __riscos
        if (gmode&MDUSEANSI)
	  _swix(OS_Write0,_IN(0),buf);
	else
	{
	  /* Do nothing */
	}
#else
	putpad(buf);
#endif
    }
#ifndef __riscos
    else if(_setaf)
      putpad(tparm(_setaf, color));
    else if(_setf)
      putpad(tparm(_setf, color));
    else if(_scp){
	/* set color pair method */
    }
#endif

    return(0);
}


static int
tbgcolor(color)
    int color;
{
    if(!_color_inited)
      tinitcolor();

    if(!_color_inited)
      return(-1);

    if((ANSI8_COLOR()  && (color < 0 || color >= 8)) ||
       (ANSI16_COLOR() && (color < 0 || color >= 16)) ||
       (!ANSI_COLOR()  && (color < 0 || color >= _colors)))
      return(-1);

    if(ANSI_COLOR()){
	char buf[10];

	if(color < 8)
	  sprintf(buf, "\033[4%cm",  color + '0');
	else
	  sprintf(buf, "\033[10%cm", (color-8) + '0');

#ifdef __riscos
        if (gmode&MDUSEANSI)
	  _swix(OS_Write0,_IN(0),buf);
	else
	{
	  /* Do nothing */
	}
#else
	putpad(buf);
#endif
    }
#ifndef __riscos
    else if(_setab)
      putpad(tparm(_setab, color));
    else if(_setb)
      putpad(tparm(_setb, color));
    else if(_scp){
	/* set color pair method */
    }
#endif

    return(0);
}



/*
 * We're not actually using the RGB value other than as a string which
 * maps into the color.
 * In fact, on some systems color 1 and color 4 are swapped, and color 3
 * and color 6 are swapped. So don't believe the RGB values.
 * Still, it feels right to have them be the canonical values, so we do that.
 *
 * More than one "name" can map to the same "canonical_name".
 * More than one "name" can map to the same "val".
 * The "val" for a "name" and for its "canonical_name" are the same.
 */
static struct color_table *
init_color_table()
{
    struct color_table *ct = NULL, *t;
    int                 i, size, count;
    char                colorname[12];

    count = pico_count_in_color_table();

    /*
     * Special case: If table is of size 8 we use a table of size 16 instead
     * and map the 2nd eight into the first 8 so that both 8 and 16 color
     * terminals can be used with the same pinerc and colors in the 2nd
     * 8 may be useful. We could make this more general and always map
     * colors larger than our current # of colors by mapping it modulo
     * the color table size. The only problem with this is that it would
     * take a boatload of programming to convert what we have now into that,
     * and it is highly likely that no one would ever use anything other
     * than the 8/16 case. So we'll stick with that special case for now.
     */
    if(count == 8)
      size = 16;
    else
      size = count;

    if(size > 0 && size < 256){
	ct = (struct color_table *)malloc((size+1)*sizeof(struct color_table));
	if(ct)
	  memset(ct, 0, (size+1) * sizeof(struct color_table));

	for(i = 0, t = ct; t && i < size; i++, t++){
	    t->val = i % count;

	    switch(i){
	      case COL_BLACK:
		strcpy(colorname, "black");
		break;
	      case COL_RED:
		strcpy(colorname, "red");
		break;
	      case COL_GREEN:
		strcpy(colorname, "green");
		break;
	      case COL_YELLOW:
		strcpy(colorname, "yellow");
		break;
	      case COL_BLUE:
		strcpy(colorname, "blue");
		break;
	      case COL_MAGENTA:
		strcpy(colorname, "magenta");
		break;
	      case COL_CYAN:
		strcpy(colorname, "cyan");
		break;
	      case COL_WHITE:
		strcpy(colorname, "white");
		break;
	      default:
		sprintf(colorname, "color%03.3d", i);
		break;
	    }

	    t->namelen = strlen(colorname);
	    t->name = (char *)malloc((t->namelen+1) * sizeof(char));
	    if(t->name)
	      strcpy(t->name, colorname);

	    /*
	     * For an 8 color terminal, canonical black == black, but
	     * canonical red == color009, etc.
	     *
	     * This is weird. What we have is 16-color xterms where color 0
	     * is black (fine, that matches) colors 1 - 7 are called
	     * red, ..., white but they are set at 2/3rds brightness so that
	     * bold can be brighter. Those 7 colors suck. Color 8 is some
	     * sort of gray, colors 9 - 15 are real red, ..., white. On other
	     * 8 color terminals and in PC-Pine, we want to equate color 0
	     * with color 0 on 16-color, but color 1 (red) is really the
	     * same as the 16-color color 9. So color 9 - 15 are the real
	     * colors on a 16-color terminal and colors 1 - 7 are the real
	     * colors on an 8-color terminal. We make that work by mapping
	     * the 2nd eight into the first eight when displaying on an
	     * 8-color terminal, and by using the canonical_name when
	     * we modify and write out a color. The canonical name is set
	     * to the "real* color (0, 9, 10, ..., 15 on 16-color terminal).
	     */
	    if(size == count || i == 0 || i > 7){
		t->canonical_name = (char *)malloc((t->namelen+1)*sizeof(char));
		strcpy(t->canonical_name, colorname);
	    }
	    else{
		t->canonical_name = (char *)malloc(9*sizeof(char));
		sprintf(t->canonical_name, "color%03.3d", i+8);
	    }

	    t->rgb = (char *)malloc((RGBLEN+1) * sizeof(char));
	    if(t->rgb){
		if(count == 8){
		  switch(i){
		    case COL_BLACK:
		    case 8:
		      strcpy(t->rgb, "000,000,000");
		      break;
		    case COL_RED:
		    case 9:
		      strcpy(t->rgb, "255,000,000");
		      break;
		    case COL_GREEN:
		    case 10:
		      strcpy(t->rgb, "000,255,000");
		      break;
		    case COL_YELLOW:
		    case 11:
		      strcpy(t->rgb, "255,255,000");
		      break;
		    case COL_BLUE:
		    case 12:
		      strcpy(t->rgb, "000,000,255");
		      break;
		    case COL_MAGENTA:
		    case 13:
		      strcpy(t->rgb, "255,000,255");
		      break;
		    case COL_CYAN:
		    case 14:
		      strcpy(t->rgb, "000,255,255");
		      break;
		    case COL_WHITE:
		    case 15:
		      strcpy(t->rgb, "255,255,255");
		      break;
		    default:
		      /*
		       * These aren't really used as RGB values, just as
		       * strings for the lookup. We don't know how to
		       * convert to any RGB, we just know how to output
		       * the escape sequences. So it doesn't matter that
		       * the numbers in the sprintf below are too big.
		       * We are using the fact that all rgb values start with
		       * a digit or space and color names don't in the lookup
		       * routines below.
		       */
		      sprintf(t->rgb, "%d,%d,%d", 256+i, 256+i, 256+i);
		      break;
		  }
		}
		else if(count == 16){
		  /*
		   * This set of RGB values seems to come close to describing
		   * what a 16-color xterm gives you.
		   */
		  switch(i){
		    case COL_BLACK:
		      strcpy(t->rgb, "000,000,000");
		      break;
		    case COL_RED:		/* actually dark red */
		      strcpy(t->rgb, "174,000,000");
		      break;
		    case COL_GREEN:		/* actually dark green */
		      strcpy(t->rgb, "000,174,000");
		      break;
		    case COL_YELLOW:		/* actually dark yellow */
		      strcpy(t->rgb, "174,174,000");
		      break;
		    case COL_BLUE:		/* actually dark blue */
		      strcpy(t->rgb, "000,000,174");
		      break;
		    case COL_MAGENTA:		/* actually dark magenta */
		      strcpy(t->rgb, "174,000,174");
		      break;
		    case COL_CYAN:		/* actually dark cyan */
		      strcpy(t->rgb, "000,174,174");
		      break;
		    case COL_WHITE:		/* actually light gray */
		      strcpy(t->rgb, "174,174,174");
		      break;
		    case 8:			/* dark gray */
		      strcpy(t->rgb, "064,064,064");
		      break;
		    case 9:			/* red */
		      strcpy(t->rgb, "255,000,000");
		      break;
		    case 10:			/* green */
		      strcpy(t->rgb, "000,255,000");
		      break;
		    case 11:			/* yellow */
		      strcpy(t->rgb, "255,255,000");
		      break;
		    case 12:			/* blue */
		      strcpy(t->rgb, "000,000,255");
		      break;
		    case 13:			/* magenta */
		      strcpy(t->rgb, "255,000,255");
		      break;
		    case 14:			/* cyan */
		      strcpy(t->rgb, "000,255,255");
		      break;
		    case 15:			/* white */
		      strcpy(t->rgb, "255,255,255");
		      break;
		    default:
		      sprintf(t->rgb, "%d,%d,%d", 256+i, 256+i, 256+i);
		      break;
		  }
		}
		else{
		  switch(i){
		    case COL_BLACK:
		      strcpy(t->rgb, "000,000,000");
		      break;
		    case COL_RED:
		      strcpy(t->rgb, "255,000,000");
		      break;
		    case COL_GREEN:
		      strcpy(t->rgb, "000,255,000");
		      break;
		    case COL_YELLOW:
		      strcpy(t->rgb, "255,255,000");
		      break;
		    case COL_BLUE:
		      strcpy(t->rgb, "000,000,255");
		      break;
		    case COL_MAGENTA:
		      strcpy(t->rgb, "255,000,255");
		      break;
		    case COL_CYAN:
		      strcpy(t->rgb, "000,255,255");
		      break;
		    case COL_WHITE:
		      strcpy(t->rgb, "255,255,255");
		      break;
		    default:
		      sprintf(t->rgb, "%d,%d,%d", 256+i, 256+i, 256+i);
		      break;
		  }
		}
	    }
	}
    }

    return(ct);
}


/*
 * Map from integer color value to canonical color name.
 */
char *
colorx(color)
    int color;
{
    struct color_table *ct;
    static char         cbuf[12];

    /* before we get set up, we still need to use this a bit */
    if(!color_tbl){
	switch(color){
	  case COL_BLACK:
	    return("black");
	  case COL_RED:
	    return("red");
	  case COL_GREEN:
	    return("green");
	  case COL_YELLOW:
	    return("yellow");
	  case COL_BLUE:
	    return("blue");
	  case COL_MAGENTA:
	    return("magenta");
	  case COL_CYAN:
	    return("cyan");
	  case COL_WHITE:
	    return("white");
	  default:
	    sprintf(cbuf, "color%03.3d", color);
	    return(cbuf);
	}
    }

    for(ct = color_tbl; ct->name; ct++)
      if(ct->val == color)
        break;

    if(ct->name)
      return(ct->canonical_name);

    /* not supposed to get here */
    sprintf(cbuf, "color%03.3d", color);
    return(cbuf);
}


/*
 * Argument is a color name which could be an RGB string, a name like "blue",
 * or a name like "color011".
 *
 * Returns a pointer to the canonical name of the color.
 */
char *
color_to_canonical_name(s)
    char *s;
{
    struct color_table *ct;

    if(!s || !color_tbl)
      return(NULL);

    if(*s == ' ' || isdigit(*s)){
	/* check for rgb string instead of name */
	for(ct = color_tbl; ct->rgb; ct++)
	  if(!strncmp(ct->rgb, s, RGBLEN))
	    break;
    }
    else{
	for(ct = color_tbl; ct->name; ct++)
	  if(!struncmp(ct->name, s, ct->namelen))
	    break;
    }

    if(ct->name)
      return(ct->canonical_name);

    return("");
}


/*
 * Argument is the name of a color or an RGB value that we recognize.
 * The table should be set up so that the val returned is the same whether
 * or not we choose the canonical name.
 *
 * Returns the integer value for the color.
 */
static int
color_to_val(s)
    char *s;
{
    struct color_table *ct;

    if(!s || !color_tbl)
      return(-1);

    if(*s == ' ' || isdigit(*s)){
	/* check for rgb string instead of name */
	for(ct = color_tbl; ct->rgb; ct++)
	  if(!strncmp(ct->rgb, s, RGBLEN))
	    break;
    }
    else{
	for(ct = color_tbl; ct->name; ct++)
	  if(!struncmp(ct->name, s, ct->namelen))
	    break;
    }

    if(ct->name)
      return(ct->val);
    else
      return(-1);
}


static void
free_color_table(ctbl)
    struct color_table **ctbl;
{
    struct color_table *t;

    if(ctbl && *ctbl){
	for(t = *ctbl; t->name; t++){
	    if(t->name){
		free(t->name);
		t->name = NULL;
	    }

	    if(t->canonical_name){
		free(t->canonical_name);
		t->canonical_name = NULL;
	    }

	    if(t->rgb){
		free(t->rgb);
		t->rgb = NULL;
	    }
	}

	free(*ctbl);
	*ctbl = NULL;
    }
}


int
pico_count_in_color_table()
{
    return(ANSI8_COLOR() ? 8 : ANSI16_COLOR() ? 16 : _colors);
}


void
pico_nfcolor(s)
    char *s;
{
    if(_nfcolor){
	free(_nfcolor);
	_nfcolor = NULL;
    }

    if(s){
	_nfcolor = (char *)malloc(strlen(s)+1);
	if(_nfcolor)
	  strcpy(_nfcolor, s);

	if(the_normal_color)
	  strcpy(the_normal_color->fg, _nfcolor);
    }
    else if(the_normal_color)
      free_color_pair(&the_normal_color);
}


void
pico_nbcolor(s)
    char *s;
{
    if(_nbcolor){
	free(_nbcolor);
	_nbcolor = NULL;
    }

    if(s){
	_nbcolor = (char *)malloc(strlen(s)+1);
	if(_nbcolor)
	  strcpy(_nbcolor, s);

	if(the_normal_color)
	  strcpy(the_normal_color->bg, _nbcolor);
    }
    else if(the_normal_color)
      free_color_pair(&the_normal_color);
}

void
pico_rfcolor(s)
    char *s;
{
    if(_rfcolor){
	free(_rfcolor);
	_rfcolor = NULL;
    }

    if(s){
	_rfcolor = (char *)malloc(strlen(s)+1);
	if(_rfcolor)
	  strcpy(_rfcolor, s);

	if(the_rev_color)
	  strcpy(the_rev_color->fg, _rfcolor);
    }
    else if(the_rev_color)
      free_color_pair(&the_rev_color);
}

void
pico_rbcolor(s)
    char *s;
{
    if(_rbcolor){
	free(_rbcolor);
	_rbcolor = NULL;
    }

    if(s){
	_rbcolor = (char *)malloc(strlen(s)+1);
	if(_rbcolor)
	  strcpy(_rbcolor, s);

	if(the_rev_color)
	  strcpy(the_rev_color->bg, _rbcolor);
    }
    else if(the_rev_color)
      free_color_pair(&the_rev_color);
}

int
pico_hascolor()
{
    if(!_color_inited)
      tinitcolor();

    return(_color_inited);
}

int
pico_usingcolor()
{
    return(_using_color && pico_hascolor());
}

void
pico_toggle_color(on)
    int on;
{
    if(on){
	if(pico_hascolor())
	  _using_color = 1;
    }
    else{
	_using_color = 0;
	if(_color_inited){
	    _color_inited = 0;
	    if(!panicking)
	      free_color_table(&color_tbl);

	    if(ANSI_COLOR())
#ifdef __riscos
              if (gmode&MDUSEANSI)
	        _swix(OS_Write0,_IN(0),"\033[39;49m");
	      else
	      {
	        /* Do nothing */
	      }
#else
	      putpad("\033[39;49m");
#endif
	    else{
#ifndef __riscos
		if(_op)
		  putpad(_op);
		if(_oc)
		  putpad(_oc);
#endif
	    }
	}
    }
}

unsigned
pico_get_color_options()
{
    return(color_options);
}

/*
 * Absolute set of options. Caller has to OR things together and so forth.
 */
void
pico_set_color_options(flags)
    unsigned flags;
{
    color_options = flags;
}

void
pico_endcolor()
{
    pico_toggle_color(0);
    if(panicking)
      return;

    if(_nfcolor){
	free(_nfcolor);
	_nfcolor = NULL;
    }

    if(_nbcolor){
	free(_nbcolor);
	_nbcolor = NULL;
    }

    if(_rfcolor){
	free(_rfcolor);
	_rfcolor = NULL;
    }

    if(_rbcolor){
	free(_rbcolor);
	_rbcolor = NULL;
    }

    if(_last_fg_color){
	free(_last_fg_color);
	_last_fg_color = NULL;
    }

    if(_last_bg_color){
	free(_last_bg_color);
	_last_bg_color = NULL;
    }

    if(the_rev_color)
      free_color_pair(&the_rev_color);

    if(the_normal_color)
      free_color_pair(&the_normal_color);
}

void
pico_set_nfg_color()
{
    if(_nfcolor)
      (void)pico_set_fg_color(_nfcolor);
}

void
pico_set_nbg_color()
{
    if(_nbcolor)
      (void)pico_set_bg_color(_nbcolor);
}

void
pico_set_normal_color()
{
    if(!_nfcolor || !_nbcolor ||
       !pico_set_fg_color(_nfcolor) || !pico_set_bg_color(_nbcolor)){
	(void)pico_set_fg_color(colorx(DEFAULT_NORM_FORE));
	(void)pico_set_bg_color(colorx(DEFAULT_NORM_BACK));
    }
}

/*
 * If inverse is a color, returns a pointer to that color.
 * If not, returns NULL.
 *
 * NOTE: Don't free this!
 */
COLOR_PAIR *
pico_get_rev_color()
{
    if(pico_usingcolor() && _rfcolor && _rbcolor &&
       pico_is_good_color(_rfcolor) && pico_is_good_color(_rbcolor)){
	if(!the_rev_color)
	  the_rev_color = new_color_pair(_rfcolor, _rbcolor);

	return(the_rev_color);
    }
    else
      return(NULL);
}

/*
 * Returns a pointer to the normal color.
 *
 * NOTE: Don't free this!
 */
COLOR_PAIR *
pico_get_normal_color()
{
    if(pico_usingcolor() && _nfcolor && _nbcolor &&
       pico_is_good_color(_nfcolor) && pico_is_good_color(_nbcolor)){
	if(!the_normal_color)
	  the_normal_color = new_color_pair(_nfcolor, _nbcolor);

	return(the_normal_color);
    }
    else
      return(NULL);
}


/*
 * Just like pico_set_color except it doesn't set the color, it just
 * returns the value. Assumes def of PSC_NONE, since otherwise we always
 * succeed and don't need to call this.
 */
int
pico_is_good_colorpair(cp)
    COLOR_PAIR *cp;
{
    if(!cp || (!pico_is_good_color(cp->fg) || !pico_is_good_color(cp->bg)))
      return(FALSE);

    return(TRUE);
}


COLOR_PAIR *
pico_set_colorp(col, flags)
    COLOR_PAIR *col;
    int         flags;
{
    return(pico_set_colors((char *) (col ? col->fg : NULL),
			   (char *) (col ? col->bg : NULL), flags));
}


/*
 * Sets color to (fg,bg).
 * Flags == PSC_NONE  No alternate default if fg,bg fails.
 *       == PSC_NORM  Set it to Normal color on failure.
 *       == PSC_REV   Set it to Reverse color on failure.
 *
 * If flag PSC_RET is set, returns an allocated copy of the previous
 * color pair, otherwise returns NULL.
 */
COLOR_PAIR *
pico_set_colors(fg, bg, flags)
    char *fg, *bg;
    int   flags;
{
    int         uc;
    COLOR_PAIR *cp = NULL, *rev = NULL;

    if(flags & PSC_RET)
      cp = pico_get_cur_color();

    if(fg && !strcmp(fg, END_PSEUDO_REVERSE)){
	EndInverse();
	if(cp)
	  free_color_pair(&cp);
    }
    else if(!((uc=pico_usingcolor()) && fg && bg &&
	      pico_set_fg_color(fg) && pico_set_bg_color(bg))){

	if(uc && flags & PSC_NORM)
	  pico_set_normal_color();
	else if(flags & PSC_REV){
	    if(rev = pico_get_rev_color()){
		pico_set_fg_color(rev->fg);	/* these will succeed */
		pico_set_bg_color(rev->bg);
	    }
	    else{
		StartInverse();
		if(cp){
		    strcpy(cp->fg, END_PSEUDO_REVERSE);
		    strcpy(cp->bg, END_PSEUDO_REVERSE);
		}
	    }
	}
    }

    return(cp);
}


int
pico_is_good_color(s)
    char *s;
{
    struct color_table *ct;

    if(!s || !color_tbl)
      return(FALSE);

    if(!strcmp(s, END_PSEUDO_REVERSE))
      return(TRUE);
    else if(*s == ' ' || isdigit(*s)){
	/* check for rgb string instead of name */
	for(ct = color_tbl; ct->rgb; ct++)
	  if(!strncmp(ct->rgb, s, RGBLEN))
	    break;
    }
    else{
	for(ct = color_tbl; ct->name; ct++)
	  if(!struncmp(ct->name, s, ct->namelen))
	    break;
    }

    return(ct->name ? TRUE : FALSE);
}


/*
 * Return TRUE on success.
 */
int
pico_set_fg_color(s)
    char *s;
{
    int val;

    if(!s || !color_tbl)
      return(FALSE);

    if(!strcmp(s, END_PSEUDO_REVERSE)){
	EndInverse();
	return(TRUE);
    }

    if((val = color_to_val(s)) >= 0){
	/* already set correctly */
	if(!_force_fg_color_change && _last_fg_color &&
	   !strcmp(_last_fg_color,colorx(val)))
	  return(TRUE);

	_force_fg_color_change = 0;
	if(_last_fg_color){
	    free(_last_fg_color);
	    _last_fg_color = NULL;
	}

	if(_last_fg_color = (char *)malloc(strlen(colorx(val))+1))
	  strcpy(_last_fg_color, colorx(val));

	tfgcolor(val);
	return(TRUE);
    }
    else
      return(FALSE);
}


int
pico_set_bg_color(s)
    char *s;
{
    int val;

    if(!s || !color_tbl)
      return(FALSE);

    if(!strcmp(s, END_PSEUDO_REVERSE)){
	EndInverse();
	return(TRUE);
    }

    if((val = color_to_val(s)) >= 0){
	/* already set correctly */
	if(!_force_bg_color_change && _last_bg_color &&
	   !strcmp(_last_bg_color,colorx(val)))
	  return(TRUE);

	_force_bg_color_change = 0;
	if(_last_bg_color){
	    free(_last_bg_color);
	    _last_bg_color = NULL;
	}

	if(_last_bg_color = (char *)malloc(strlen(colorx(val))+1))
	  strcpy(_last_bg_color, colorx(val));

	tbgcolor(val);
	return(TRUE);
    }
    else
      return(FALSE);
}


/*
 * Return a pointer to an rgb string for the input color. The output is 11
 * characters long and looks like rrr,ggg,bbb.
 *
 * Args    colorName -- The color to convert to ascii rgb.
 *
 * Returns  Pointer to a static buffer containing the rgb string.
 */
char *
color_to_asciirgb(colorName)
    char *colorName;
{
    static char c_to_a_buf[RGBLEN+1];

    struct color_table *ct;

    for(ct = color_tbl; ct && ct->name; ct++)
      if(!strucmp(ct->name, colorName))
	break;

    if(ct && ct->name)
      strcpy(c_to_a_buf, ct->rgb);
    else{
	int l;

	/*
	 * If we didn't find the color we're in a bit of trouble. This
	 * most likely means that the user is using the same pinerc on
	 * two terminals, one with more colors than the other. We didn't
	 * find a match because this color isn't present on this terminal.
	 * Since the return value of this function is assumed to be
	 * RGBLEN long, we'd better make it that long.
	 * It still won't work correctly because colors will be screwed up,
	 * but at least the embedded colors in filter.c will get properly
	 * sucked up when they're encountered.
	 */
	strncpy(c_to_a_buf, "xxxxxxxxxxx", RGBLEN);  /* RGBLEN is 11 */
	l = strlen(colorName);
	strncpy(c_to_a_buf, colorName, (l < RGBLEN) ? l : RGBLEN);
	c_to_a_buf[RGBLEN] = '\0';
    }

    return(c_to_a_buf);
}

char *
pico_get_last_fg_color()
{
    char *ret = NULL;

    if(_last_fg_color)
      if(ret = (char *)malloc(strlen(_last_fg_color)+1))
	strcpy(ret, _last_fg_color);

    return(ret);
}

char *
pico_get_last_bg_color()
{
    char *ret = NULL;

    if(_last_bg_color)
      if(ret = (char *)malloc(strlen(_last_bg_color)+1))
	strcpy(ret, _last_bg_color);

    return(ret);
}


COLOR_PAIR *
pico_get_cur_color()
{
    return(new_color_pair(_last_fg_color, _last_bg_color));
}


COLOR_PAIR *
new_color_pair(fg, bg)
    char *fg, *bg;
{
    COLOR_PAIR *ret;

    if((ret = (COLOR_PAIR *)malloc(sizeof(*ret))) != NULL){
	memset(ret, 0, sizeof(*ret));
	if(fg){
	    strncpy(ret->fg, fg, MAXCOLORLEN);
	    ret->fg[MAXCOLORLEN] = '\0';
	}

	if(bg){
	    strncpy(ret->bg, bg, MAXCOLORLEN);
	    ret->bg[MAXCOLORLEN] = '\0';
	}
    }

    return(ret);
}


void
free_color_pair(cp)
    COLOR_PAIR **cp;
{
    if(cp && *cp){
	free(*cp);
	*cp = NULL;
    }
}


/*
 * alt_editor - fork off an alternate editor for mail message composition
 *              if one is configured and passed from pine.  If not, only
 *              ask for the editor if advanced user flag is set, and
 *              suggest environment's EDITOR value as default.
 */
alt_editor(f, n)
{
#ifndef __riscos
    char   eb[NLINE];				/* buf holding edit command */
    char   *fn;					/* tmp holder for file name */
    char   result[128];				/* result string */
    char   prmpt[128];
    int	   child, pid, i, done = 0, ret = 0, rv;
    SigType (*ohup) SIG_PROTO((int));
    SigType (*oint) SIG_PROTO((int));
    SigType (*osize) SIG_PROTO((int));
#if	defined(HAVE_WAIT_UNION)
    union  wait stat;
#ifndef	WIFEXITED
#define	WIFEXITED(X)	(!(X).w_termsig)	/* nonzero if child killed */
#endif
#ifndef	WEXITSTATUS
#define	WEXITSTATUS(X)	X.w_retcode		/* child's exit value */
#endif
#else
    int    stat;
#ifndef	WIFEXITED
#define	WIFEXITED(X)	(!((X) & 0xff))		/* low bits, child killed */
#endif
#ifndef	WEXITSTATUS
#define	WEXITSTATUS(X)	((X) >> 8)		/* high bits, exit value */
#endif
#endif

#ifndef	PATH_MAX
#define	PATH_MAX	NLINE
#endif

    if(gmode&MDSCUR){
	emlwrite("Alternate %s not available in restricted mode",
		 f ? "speller" : "editor");
	return(-1);
    }

    strcpy(result, "Alternate %s complete.");

    if(f){
	if(alt_speller)
	  strcpy(eb, alt_speller);
	else
	  return(-1);
    }
    else if(Pmaster == NULL){
	return(-1);
    }
    else{
	eb[0] = '\0';

	if(Pmaster->alt_ed){
	    char **lp, *wsp, *path, fname[PATH_MAX+1];
	    int	   c;

	    for(lp = Pmaster->alt_ed; *lp && **lp; lp++){
		if(wsp = strpbrk(*lp, " \t")){
		    c = *wsp;
		    *wsp = '\0';
		}

		if(strchr(*lp, '/')){
		    rv = fexist(*lp, "x", (off_t *)NULL);
		}
		else{
		    if(!(path = getenv("PATH")))
		      path = ":/bin:/usr/bin";

		    rv = ~FIOSUC;
		    while(rv != FIOSUC && *path && pathcat(fname, &path, *lp))
		      rv = fexist(fname, "x", (off_t *)NULL);
		}

		if(wsp)
		  *wsp = c;

		if(rv == FIOSUC){
		    strcpy(eb, *lp);
		    break;
		}
	    }
	}

	if(!eb[0]){
	    if(!(gmode&MDADVN)){
		emlwrite("\007Unknown Command",NULL);
		return(-1);
	    }

#ifdef __riscos
	    if(getenv("Pilot$Editor"))
	      strcpy(eb, (char *)getenv("Pilot$Editor"));
	    else
#endif
	    if(getenv("EDITOR"))
	      strcpy(eb, (char *)getenv("EDITOR"));
	    else
	      *eb = '\0';

	    while(!done){
		pid = mlreplyd("Which alternate editor ? ", eb,
			       NLINE, QDEFLT, NULL);
		switch(pid){
		  case ABORT:
		    curwp->w_flag |= WFMODE;
		    return(-1);
		  case HELPCH:
		    emlwrite("no alternate editor help yet", NULL);

/* take sleep and break out after there's help */
		    sleep(3);
		    break;
		  case (CTRL|'L'):
		    sgarbf = TRUE;
		    update();
		    break;
		  case TRUE:
		  case FALSE:			/* does editor exist ? */
		    if(*eb == '\0'){		/* leave silently? */
			mlerase();
			curwp->w_flag |= WFMODE;
			return(-1);
		    }

		    done++;
		    break;
		    default:
		    break;
		}
	    }
	}
    }

    if((fn=writetmp(1, NULL)) == NULL){		/* get temp file */
	emlwrite("Problem writing temp file for alt editor", NULL);
	return(-1);
    }

    strcat(eb, " ");
    strcat(eb, fn);


    for(i = 0; i <= ((Pmaster) ? term.t_nrow : term.t_nrow - 1); i++){
	movecursor(i, 0);
	if(!i){
	    fputs("Invoking alternate ", stdout);
	    fputs(f ? "speller..." : "editor...", stdout);
	}

	peeol();
    }

    (*term.t_flush)();
    if(Pmaster)
      (*Pmaster->tty_fix)(0);
    else
      vttidy();

#ifdef	SIGCHLD
    if(Pmaster){
	/*
	 * The idea here is to keep any mail connections our caller
	 * may have open while our child's out running around...
	 */
	pico_child_done = pico_child_jmp_ok = 0;
	(void) signal(SIGCHLD,  child_handler);
    }
#endif

    if((child = fork()) > 0){		/* wait for the child to finish */
	ohup = signal(SIGHUP, SIG_IGN);	/* ignore signals for now */
	oint = signal(SIGINT, SIG_IGN);
#ifdef	RESIZING
        osize = signal(SIGWINCH, SIG_IGN);
#endif

#ifdef	SIGCHLD
	if(Pmaster){
	    while(!pico_child_done){
		(*Pmaster->newmail)(0, 0);
		if(!pico_child_done){
		    if(setjmp(pico_child_state) == 0){
			pico_child_jmp_ok = 1;
			sleep(600);
		    }
		    else
		      our_sigunblock(SIGCHLD);
		}

		pico_child_jmp_ok = 0;
	    }
	}
#endif

	while((pid = (int) wait(&stat)) != child)
	  ;

	signal(SIGHUP, ohup);	/* restore signals */
	signal(SIGINT, oint);
#ifdef	RESIZING
        signal(SIGWINCH, osize);
#endif

	/*
	 * Report child's unnatural or unhappy exit...
	 */
	if(WIFEXITED(stat) && WEXITSTATUS(stat) == 0)
	  strcpy(result, "Alternate %s done");
	else {
	  sprintf(result, "Alternate %%s terminated abnormally (%d)",
		  WIFEXITED(stat) ? WEXITSTATUS(stat) : -1);
	  if(f)
	    ret = -1;
	  else{
	      sprintf(prmpt, "Alt editor failed, use file %.20s of size %%ld chars anyway", fn);
	      ret = -2;
	  }
	}
    }
    else if(child == 0){		/* spawn editor */
	signal(SIGHUP, SIG_DFL);	/* let editor handle signals */
	signal(SIGINT, SIG_DFL);
#ifdef	RESIZING
        signal(SIGWINCH, SIG_DFL);
#endif
#ifdef	SIGCHLD
	(void) signal(SIGCHLD,  SIG_DFL);
#endif
	if(execl("/bin/sh", "sh", "-c", eb, 0) < 0)
	  exit(-1);
    }
    else {				/* error! */
	sprintf(result, "\007Can't fork %%s: %s", errstr(errno));
	ret = -1;
    }

#ifdef	SIGCHLD
    (void) signal(SIGCHLD,  SIG_DFL);
#endif

    if(Pmaster)
      (*Pmaster->tty_fix)(1);

    /*
     * Editor may have set a hibit, we don't know. Assume it did.
     */
    if(!f && Pmaster && Pmaster->hibit_entered)
     *Pmaster->hibit_entered = 1;

    /*
     * replace edited text with new text
     */
    curbp->b_flag &= ~BFCHG;		/* make sure old text gets blasted */

    if(ret == -2){
	off_t filesize;
	char  prompt[128];

	rv = fexist(fn, "r", &filesize);
	if(rv == FIOSUC && filesize > 0){
	    sprintf(prompt, prmpt, (long) filesize);
	    /* clear bottom 3 rows */
	    pclear(term.t_nrow-2, term.t_nrow+1);
	    i = mlyesno(prompt, FALSE);
	    if(i == TRUE){
		ret = 0;
		strcpy(result, "OK, alternate %s done");
	    }
	    else
	      ret = -1;
	}
	else
	  ret = -1;
    }

    if(ret == 0)
      readin(fn, 0, 0);			/* read new text overwriting old */

    unlink(fn);				/* blast temp file */
    curbp->b_flag |= BFCHG;		/* mark dirty for packbuf() */
    ttopen();				/* reset the signals */
    pico_refresh(0, 1);			/* redraw */
    update();
    emlwrite(result, f ? "speller" : "editor");
    return(ret);
#else
    return(-1);
#endif
}



int
pathcat(buf, path, file)
    char *buf, **path, *file;
{
    register int n = 0;

    while(**path && **path != ':'){
	if(n++ > PATH_MAX)
	  return(FALSE);

	*buf++ = *(*path)++;
    }

    if(n){
	if(n++ > PATH_MAX)
	  return(FALSE);

	*buf++ = '/';
    }

    while(*buf = *file++){
	if(n++ > PATH_MAX)
	  return(FALSE);

	buf++;
    }

    if(**path == ':')
      (*path)++;

    return(TRUE);
}




/*
 *  bktoshell - suspend and wait to be woken up
 */
int
bktoshell()		/* suspend MicroEMACS and wait to wake up */
{
#ifdef	SIGTSTP
    if(!(gmode&MDSSPD)){
	emlwrite("\007Unknown command: ^Z", NULL);
	return(0);
    }

    if(Pmaster){
	if(!Pmaster->suspend){
	    emlwrite("\007Unknown command: ^Z", NULL);
	    return(0);
	}

	if((*Pmaster->suspend)() == NO_OP_COMMAND){
	    int rv;

	    if(km_popped){
		term.t_mrow = 2;
		curwp->w_ntrows -= 2;
	    }

	    clearcursor();
	    mlerase();
	    rv = (*Pmaster->showmsg)('x');
	    ttresize();
	    picosigs();
	    if(rv)		/* Did showmsg corrupt the display? */
	      pico_refresh(0, 1);	/* Yes, repaint */

	    mpresf = 1;
	    if(km_popped){
		term.t_mrow = 0;
		curwp->w_ntrows += 2;
	    }
	}
	else{
	    ttresize();
	    pclear(0, term.t_nrow);
	    pico_refresh(0, 1);
	}

	return(1);
    }

    if(gmode&MDSPWN){
	char *shell;

	vttidy();
	movecursor(0, 0);
	(*term.t_eeop)();
	printf("\n\n\nUse \"exit\" to return to Pi%s\n",
	       (gmode & MDBRONLY) ? "lot" : "co");
	system((shell = (char *)getenv("SHELL")) ? shell : "/bin/csh");
	rtfrmshell SIG_PROTO((dummy));	/* fixup tty */
    }
    else {
	movecursor(term.t_nrow-1, 0);
	peeol();
	movecursor(term.t_nrow, 0);
	peeol();
	movecursor(term.t_nrow, 0);
	printf("\n\n\nUse \"fg\" to return to Pi%s\n",
	       (gmode & MDBRONLY) ? "lot" : "co");
	ttclose();
	movecursor(term.t_nrow, 0);
	peeol();
	(*term.t_flush)();

	signal(SIGCONT, rtfrmshell);	/* prepare to restart */
	signal(SIGTSTP, SIG_DFL);			/* prepare to stop */
	kill(0, SIGTSTP);
    }

    return(1);
#else
    return 0;
#endif
}


/*
 * rtfrmshell - back from shell, fix modes and return
 */
SigType
rtfrmshell SIG_PROTO((int sig))
{
#ifdef	SIGCONT
    signal(SIGCONT, SIG_DFL);
    ttopen();
    ttresize();
    pclear(0, term.t_nrow);
    pico_refresh(0, 1);
#endif
}



/*
 * do_hup_signal - jump back in the stack to where we can handle this
 */
SigType
do_hup_signal SIG_PROTO((int sig))
{
#ifndef __riscos
    signal(SIGHUP,  SIG_IGN);			/* ignore further SIGHUP's */
#endif
    signal(SIGTERM, SIG_IGN);			/* ignore further SIGTERM's */
    if(Pmaster){
	extern jmp_buf finstate;

	longjmp(finstate, COMP_GOTHUP);
    }
    else{
	/*
	 * if we've been interrupted and the buffer is changed,
	 * save it...
	 */
	if(anycb() == TRUE){			/* time to save */
	    if(curbp->b_fname[0] == '\0'){	/* name it */
#ifdef __riscos
		strcpy(curbp->b_fname, "pico/save");
#else
		strcpy(curbp->b_fname, "pico.save");
#endif
	    }
	    else{
#ifdef __riscos
		strcat(curbp->b_fname, "/save");
#else
		strcat(curbp->b_fname, ".save");
#endif
	    }
	    remove(curbp->b_fname);
	    writeout(curbp->b_fname, TRUE);
	}
	vttidy();
	exit(1);
    }
}


/*
 * big bitmap of ASCII characters allowed in a file name
 * (needs reworking for other char sets)
 */
unsigned char okinfname[32] = {
      0,    0, 			/* ^@ - ^G, ^H - ^O  */
      0,    0,			/* ^P - ^W, ^X - ^_  */
      0x80, 0x17,		/* SP - ' ,  ( - /   */
#ifdef __riscos
      0xff, 0xc0,		/*  0 - 7 ,  8 - ?   */
#else
      0xff, 0xc4,		/*  0 - 7 ,  8 - ?   */
#endif
      0x7f, 0xff,		/*  @ - G ,  H - O   */
      0xff, 0xe1,		/*  P - W ,  X - _   */
      0x7f, 0xff,		/*  ` - g ,  h - o   */
      0xff, 0xf6,		/*  p - w ,  x - DEL */
      0,    0, 			/*  > DEL   */
      0,    0,			/*  > DEL   */
      0,    0, 			/*  > DEL   */
      0,    0, 			/*  > DEL   */
      0,    0 			/*  > DEL   */
};


/*
 * fallowc - returns TRUE if c is allowable in filenames, FALSE otw
 */
fallowc(c)
char c;
{
    return(okinfname[c>>3] & 0x80>>(c&7));
}


/*
 * fexist - returns TRUE if the file exists with mode passed in m,
 *          FALSE otherwise.  By side effect returns length of file in l
 */
fexist(file, m, l)
char  *file;
char  *m;			/* files mode: r,w,rw,t or x */
off_t *l;			/* t means use lstat         */
{
#ifdef __riscos
  int obj;
  int len;
  {
    char *tail=file+strlen(file);
    if (strlen(file)>3)
    {
      if (strcmp(&tail[-2],"..")==0)
      {
        l=0;
        return FIODIR;
      }
    }
  }
  _swix(OS_File,_INR(0,1)|_OUT(0)|_OUT(4),17,file,&obj,&len);
  if (l!=NULL) *l=len; /* the length */
  switch (obj)
  {
    case 0:
     return(FIOFNF);
    case 1:
     return(FIOSUC);
    case 2:
     return(FIODIR);
    case 3:
     return(FIODIR);
  }
  return(FIOERR);
#else
    struct stat	sbuf;
    extern int lstat();
    int		(*stat_f)() = (m && *m == 't') ? lstat : stat;

    if(l)
      *l = (off_t)0;

    if((*stat_f)(file, &sbuf) < 0){
	switch(errno){
	  case ENOENT :				/* File not found */
	    return(FIOFNF);
#ifdef	ENAMETOOLONG
	  case ENAMETOOLONG :			/* Name is too long */
	    return(FIOLNG);
#endif
	  case EACCES :				/* File not found */
	    return(FIOPER);
	  default:				/* Some other error */
	    return(FIOERR);
	}
    }

    if(l)
      *l = (off_t)sbuf.st_size;

    if((sbuf.st_mode&S_IFMT) == S_IFDIR)
      return(FIODIR);
    else if(*m == 't'){
	struct stat	sbuf2;

	/*
	 * If it is a symbolic link pointing to a directory, treat
	 * it like it is a directory, not a link.
	 */
	if((sbuf.st_mode&S_IFMT) == S_IFLNK){
	    if(stat(file, &sbuf2) < 0){
		switch(errno){
		  case ENOENT :				/* File not found */
		    return(FIOSYM);			/* call it a link */
#ifdef	ENAMETOOLONG
		  case ENAMETOOLONG :			/* Name is too long */
		    return(FIOLNG);
#endif
		  case EACCES :				/* File not found */
		    return(FIOPER);
		  default:				/* Some other error */
		    return(FIOERR);
		}
	    }

	    if((sbuf2.st_mode&S_IFMT) == S_IFDIR)
	      return(FIODIR);
	}

	return(((sbuf.st_mode&S_IFMT) == S_IFLNK) ? FIOSYM : FIOSUC);
    }

    if(*m == 'r'){				/* read access? */
	if(*(m+1) == 'w')			/* and write access? */
	  return((can_access(file,READ_ACCESS)==0)
		 ? (can_access(file,WRITE_ACCESS)==0)
		    ? FIOSUC
		    : FIONWT
		 : FIONRD);
	else if(!*(m+1))			/* just read access? */
	  return((can_access(file,READ_ACCESS)==0) ? FIOSUC : FIONRD);
    }
    else if(*m == 'w' && !*(m+1))		/* write access? */
      return((can_access(file,WRITE_ACCESS)==0) ? FIOSUC : FIONWT);
    else if(*m == 'x' && !*(m+1))		/* execute access? */
      return((can_access(file,EXECUTE_ACCESS)==0) ? FIOSUC : FIONEX);
    return(FIOERR);				/* bad m arg */
#endif
}


/*
 * isdir - returns true if fn is a readable directory, false otherwise
 *         silent on errors (we'll let someone else notice the problem;)).
 */
isdir(fn, l, d)
char *fn;
long *l;
time_t *d;
{
#ifdef __riscos
  int obj;
  int len;
  {
    char *tail=fn+strlen(fn);
    if (strlen(fn)>3)
    {
      if (strcmp(&tail[-2],"..")==0)
      {
        l=0;
        return 1;
      }
    }
  }
  _swix(OS_File,_INR(0,1)|_OUT(0)|_OUT(4),17,fn,&obj,&len);
  if (l!=NULL)
    *l=len; /* the length */
  if (d)
    *d=0; /* NO time just yet!!! */
  return obj>1;
#else
    struct stat sbuf;

    if(l)
      *l = 0;

    if(stat(fn, &sbuf) < 0)
      return(0);

    if(l)
      *l = sbuf.st_size;

    if(d)
      *d = sbuf.st_mtime;

    return((sbuf.st_mode&S_IFMT) == S_IFDIR);
#endif
}


/*
 * gethomedir - returns the users home directory
 *              Note: home is malloc'd for life of pico
 */
char *
gethomedir(l)
int *l;
{
    static char *home = NULL;
    static short hlen = 0;

    if(home == NULL){
	char buf[NLINE];
#ifdef __riscos
	strcpy(buf, "@");
#else
	strcpy(buf, "~");
	fixpath(buf, NLINE);		/* let fixpath do the work! */
#endif
	hlen = strlen(buf);
	if((home = (char *)malloc((hlen + 1) * sizeof(char))) == NULL){
	    emlwrite("Problem allocating space for home dir", NULL);
	    return(0);
	}

	strcpy(home, buf);
    }

    if(l)
      *l = hlen;

    return(home);
}


/*
 * homeless - returns true if given file does not reside in the current
 *            user's home directory tree.
 */
homeless(f)
char *f;
{
    char *home;
    int   len;

    home = gethomedir(&len);
    return(strncmp(home, f, len));
}



/*
 * errstr - return system error string corresponding to given errno
 *          Note: strerror() is not provided on all systems, so it's
 *          done here once and for all.
 *
 *        Not anymore! There are now systems that won't let you use
 *        sys_nerr and sys_errlist, so it is time to switch to using
 *        strerror instead. If this doesn't work, uncomment the old stuff.
 */
char *
errstr(err)
int err;
{
    return((err >= 0) ? (char *) strerror(err) : NULL);
#ifdef OLDWAY
    return((err >= 0 && err < sys_nerr) ? (char *)sys_errlist[err] : NULL);
#endif
}



/*
 * getfnames - return all file names in the given directory in a single
 *             malloc'd string.  n contains the number of names
 */
char *
getfnames(dn, pat, n, e)
char *dn, *pat, *e;
int  *n;
{
    long           l;
    size_t	   avail, alloced, incr = 1024;
    char          *names, *np, *p;
    struct stat    sbuf;
#if	defined(ct)
    FILE          *dirp;
    char           fn[DIRSIZ+1];
#else
    DIR           *dirp;			/* opened directory */
#endif
#ifdef	USE_DIRENT
    struct dirent *dp;
#else
    struct direct *dp;
#endif

    *n = 0;

    if(stat(dn, &sbuf) < 0){
	switch(errno){
#ifdef ENOTENT
	  case ENOENT :				/* File not found */
	    if(e)
	      sprintf(e, "\007File not found: \"%s\"", dn);

	    break;
#endif
#ifdef	ENAMETOOLONG
	  case ENAMETOOLONG :			/* Name is too long */
	    if(e)
	      sprintf(e, "\007File name too long: \"%s\"", dn);

	    break;
#endif
	  default:				/* Some other error */
	    if(e)
	      sprintf(e, "\007Error getting file info: \"%s\"", dn);

	    break;
	}
	return(NULL);
    }
    else{
#ifndef MAX
#define MAX(x,y)        ((x) > (y) ? (x) : (y))
#endif
	/*
	 * We'd like to use 512 * st_blocks as an initial estimate but
	 * some systems have a stat struct with no st_blocks in it.
	 */
	avail = alloced = MAX(sbuf.st_size, incr);
	if((sbuf.st_mode&S_IFMT) != S_IFDIR){
	    if(e)
	      sprintf(e, "\007Not a directory: \"%s\"", dn);

	    return(NULL);
	}
#ifdef __riscos
        /* JRF: FIXME: I don't think we get this right at the moment - check this */
        /* This is a patch around the problem */
      l = 0;
      if((dirp=opendir(dn)) == NULL) {
        if(e)
          sprintf(e, "\007Can't open \"%s\": %s", dn, errstr(errno));
        return(NULL);
      }
      if(!pat || !*pat || !strncmp("..", pat, strlen(pat))) {
        l+=3;
      }
      while((dp = readdir(dirp)) != NULL)
        if(!pat || !*pat || !strncmp(dp->d_name, pat, strlen(pat))) {
          p = dp->d_name;
          while(*p++)
            l++;
          l++;
        }
      closedir(dirp);					/* shut down */
      avail = alloced = MAX(alloced, l);
#endif
    }

    if((names=(char *)malloc(alloced * sizeof(char))) == NULL){
	if(e)
	  sprintf(e, "\007Can't malloc space for file names");

	return(NULL);
    }

    errno = 0;
    if((dirp=opendir(dn)) == NULL){
	if(e)
	  sprintf(e, "\007Can't open \"%s\": %s", dn, errstr(errno));

	free((char *)names);
	return(NULL);
    }

    np = names;

#if	defined(ct)
    while(fread(&dp, sizeof(struct direct), 1, dirp) > 0) {
    /* skip empty slots with inode of 0 */
	if(dp.d_ino == 0)
	     continue;
	(*n)++;                     /* count the number of active slots */
	(void)strncpy(fn, dp.d_name, DIRSIZ);
	fn[14] = '\0';
	p = fn;
	while((*np++ = *p++) != '\0')
	  ;
    }
#else
#ifdef __riscos
    { /* 'parent' directory' */
      p="..";
      while(*np++ = *p++)
        avail--;
      (*n)++;                     /* count the number of active slots */
    }
#endif
    while((dp = readdir(dirp)) != NULL)
      if(!pat || !*pat || !strncmp(dp->d_name, pat, strlen(pat))){
	  (*n)++;
	  p = dp->d_name;
	  l = strlen(p);
	  while(avail < l+1){
	      char *oldnames;

	      alloced += incr;
	      avail += incr;
	      oldnames = names;
	      if((names=(char *)realloc((void *)names, alloced * sizeof(char)))
		  == NULL){
		if(e)
		  sprintf(e, "\007Can't malloc enough space for file names");

		return(NULL);
	      }

	      np = names + (np-oldnames);
	  }

	  avail -= (l+1);

	  while(*np++ = *p++)
	    ;
      }
#endif

    closedir(dirp);					/* shut down */
    return(names);
}


/*
 * fioperr - given the error number and file name, display error
 */
void
fioperr(e, f)
int  e;
char *f;
{
    switch(e){
      case FIOFNF:				/* File not found */
	emlwrite("\007File \"%s\" not found", f);
	break;
      case FIOEOF:				/* end of file */
	emlwrite("\007End of file \"%s\" reached", f);
	break;
      case FIOLNG:				/* name too long */
	emlwrite("\007File name \"%s\" too long", f);
	break;
      case FIODIR:				/* file is a directory */
	emlwrite("\007File \"%s\" is a directory", f);
	break;
      case FIONWT:
	emlwrite("\007Write permission denied: %s", f);
	break;
      case FIONRD:
	emlwrite("\007Read permission denied: %s", f);
	break;
      case FIOPER:
	emlwrite("\007Permission denied: %s", f);
	break;
      case FIONEX:
	emlwrite("\007Execute permission denied: %s", f);
	break;
      default:
	emlwrite("\007File I/O error: %s", f);
    }
}



/*
 * pfnexpand - pico's function to expand the given file name if there is
 *	       a leading '~'
 */
char *pfnexpand(fn, len)
    char  *fn;
    size_t len;
{
    struct passwd *pw;
    register char *x, *y, *z;
    char *home = NULL;
    char name[20];

#ifndef __riscos
    if(*fn == '~') {
        for(x = fn+1, y = name;
	    *x != '/' && *x != '\0' && y-name < sizeof(name)-1;
	    *y++ = *x++)
	  ;

        *y = '\0';
        if(x == fn + 1){			/* ~/ */
	    if (!(home = (char *) getenv("HOME")))
	      if (pw = getpwuid(geteuid()))
		home = pw->pw_dir;
	}
	else if(*name){				/* ~username/ */
	    if(pw = getpwnam(name))
	      home = pw->pw_dir;
	}

        if(!home || (strlen(home) + strlen(fn) >= len))
	  return(NULL);

	/* make room for expanded path */
	for(z = x + strlen(x), y = fn + strlen(x) + strlen(home);
	    z >= x;
	    *y-- = *z--)
	  ;

	/* and insert the expanded address */
	for(x = fn, y = home; *y != '\0'; *x++ = *y++)
	  ;
    }
#endif
    return(fn);
}



/*
 * fixpath - make the given pathname into an absolute path
 */
void
fixpath(name, len)
    char  *name;
    size_t len;
{
#ifndef __riscos
    register char *shft;

    /* filenames relative to ~ */
    if(!((name[0] == '/')
          || (name[0] == '.'
              && (name[1] == '/' || (name[1] == '.' && name[2] == '/'))))){
	if(Pmaster && !(gmode&MDCURDIR)
                   && (*name != '~' && strlen(name)+2 < len)){

	    if(gmode&MDTREE && strlen(name)+strlen(opertree)+1 < len){
		int off = strlen(opertree);

		for(shft = strchr(name, '\0'); shft >= name; shft--)
		  shft[off+1] = *shft;

		strncpy(name, opertree, off);
		name[off] = '/';
	    }
	    else{
		for(shft = strchr(name, '\0'); shft >= name; shft--)
		  shft[2] = *shft;

		name[0] = '~';
		name[1] = '/';
	    }
	}

	pfnexpand(name, len);
    }
#endif
}


/*
 * compresspath - given a base path and an additional directory, collapse
 *                ".." and "." elements and return absolute path (appending
 *                base if necessary).
 *
 *                returns  1 if OK,
 *                         0 if there's a problem
 *                         new path, by side effect, if things went OK
 */
compresspath(base, path, len)
char *base, *path;
int  len;
{
    register int i;
    int  depth = 0;
    char *p;
    char *stack[32];
    char  pathbuf[NLINE];

#define PUSHD(X)  (stack[depth++] = X)
#define POPD()    ((depth > 0) ? stack[--depth] : "")

#ifndef __riscos
    if(*path == '~'){
	fixpath(path, len);
	strcpy(pathbuf, path);
    }
    else if(*path != C_FILESEP)
      sprintf(pathbuf, "%s%c%s", base, C_FILESEP, path);
#else
         if (strchr(path, '$') == NULL)
      sprintf(pathbuf, "%s%c%s", (base && base[0] != '\0') ? base : "@", C_FILESEP, path);
#endif
    else
      strcpy(pathbuf, path);

    p = &pathbuf[0];
#ifdef __riscos
    /* The first component is always the top level - there is not top level separator */
    PUSHD(p);				/* push dir entry */
#endif
    for(i=0; pathbuf[i] != '\0'; i++){		/* pass thru path name */
#ifdef __riscos
	if(pathbuf[i] == '.'){
#else
	if(pathbuf[i] == '/'){
#endif
	    if(p != pathbuf)
	      PUSHD(p);				/* push dir entry */

	    p = &pathbuf[i+1];			/* advance p */
	    pathbuf[i] = '\0';			/* cap old p off */
	    continue;
	}

#ifdef __riscos
	if(pathbuf[i] == '^' &&
	   (pathbuf[i+1] == '.' || pathbuf[i+1] == '\0')) {
#else
	if(pathbuf[i] == '.'){			/* special cases! */
	    if(pathbuf[i+1] == '.' 		/* parent */
	       && (pathbuf[i+2] == '/' || pathbuf[i+2] == '\0')){
#endif
		if(!strcmp(POPD(), ""))		/* bad news! */
		  return(0);

#ifndef __riscos
		i += 2;
#else
		i += 1;
#endif
		p = (pathbuf[i] == '\0') ? "" : &pathbuf[i+1];
	    }
#ifndef __riscos
	    else if(pathbuf[i+1] == '/' || pathbuf[i+1] == '\0'){
		i++;
		p = (pathbuf[i] == '\0') ? "" : &pathbuf[i+1];
	    }
	}
#endif
    }

    if(*p != '\0')
      PUSHD(p);					/* get last element */

    path[0] = '\0';
    for(i = 0; i < depth; i++){
#ifdef __riscos
        if (i!=0)
#endif
	strcat(path, S_FILESEP);
	strcat(path, stack[i]);
    }

    return(1);					/* everything's ok */
}



#ifndef __riscos
/*
 *     Check if we can access a file in a given way
 *
 * Args: file      -- The file to check
 *       mode      -- The mode ala the access() system call, see ACCESS_EXISTS
 *                    and friends in pine.h.
 *
 * Result: returns 0 if the user can access the file according to the mode,
 *         -1 if he can't (and errno is set).
 */
int
can_access(file, mode)
    char *file;
    int   mode;
{
    return(access(file, mode));
}


/*
 * This routine is derived from BSD4.3 code,
 * Copyright (c) 1987 Regents of the University of California.
 * All rights reserved.
 */
#if defined(LIBC_SCCS) && !defined(lint)
static char sccsid[] = "@(#)mktemp.c	5.7 (Berkeley) 6/27/88";
#endif /* LIBC_SCCS and not lint */

static char *
was_nonexistent_tmp_name(as, create_it, ext)
    char *as;
    int   create_it;
    char *ext;
{
    register char  *start, *trv;
    struct stat     sbuf;
    unsigned        pid;
    static unsigned n = 0;
    int             fd, tries = 0;

    pid = ((unsigned)getpid() * 100) + n++;

    /* extra X's get set to 0's */
    for(trv = as; *trv; ++trv)
      ;

    /*
     * We should probably make the name random instead of having it
     * be the pid.
     */
    while(*--trv == 'X'){
	*trv = (pid % 10) + '0';
	pid /= 10;
    }

    /* add the extension, enough room guaranteed by caller */
    if(ext){
	strcat(as, ".");
	strcat(as, ext);
    }

    /*
     * Check for write permission on target directory; if you have
     * six X's and you can't write the directory, this will run for
     * a *very* long time.
     */
    for(start = ++trv; trv > as && *trv != '/'; --trv)
      ;

    if(*trv == '/'){
	*trv = '\0';
	if(stat(as==trv ? "/" : as, &sbuf) || !(sbuf.st_mode & S_IFDIR))
	  return((char *)NULL);

	*trv = '/';
    }
    else if (stat(".", &sbuf) == -1)
      return((char *)NULL);

    for(;;){
	/*
	 * Check with lstat to be sure we don't have
	 * a symlink. If lstat fails and no such file, then we
	 * have a winner. Otherwise, lstat shouldn't fail.
	 * If lstat succeeds, then skip it because it exists.
	 */
	if(lstat(as, &sbuf)){		/* lstat failed */
	    if(errno == ENOENT){		/* no such file, success */
		/*
		 * If create_it is set, create the file so that the
		 * evil ones don't have a chance to put something there
		 * that they can read or write before we create it
		 * ourselves.
		 */
		if(!create_it ||
		   ((fd=open(as, O_CREAT|O_EXCL|O_WRONLY,0600)) >= 0
		    && close(fd) == 0))
		  return(as);
		else if(++tries > 3)		/* open failed unexpectedly */
		  return((char *)NULL);
	    }
	    else				/* failed for unknown reason */
	      return((char *)NULL);
	}

	for(trv = start;;){
	    if(!*trv)
	      return((char *)NULL);

	    /*
	     * Change the digits from the initial values into
	     * lower case letters and try again.
	     */
	    if(*trv == 'z')
	      *trv++ = 'a';
	    else{
		if(isdigit((unsigned char)*trv))
		  *trv = 'a';
		else
		  ++*trv;

		break;
	    }
	}
    }
    /*NOTREACHED*/
}
#endif



/*
 * This routine is derived from BSD4.3 code,
 * Copyright (c) 1988 Regents of the University of California.
 * All rights reserved.
 */
#if defined(LIBC_SCCS) && !defined(lint)
static char sccsid[] = "@(#)tmpnam.c	4.5 (Berkeley) 6/27/88";
#endif /* LIBC_SCCS and not lint */
/*----------------------------------------------------------------------
      Return a unique file name in a given directory.  This is not quite
      the same as the usual tempnam() function, though it is similar.
      We want it to use the TMPDIR/TMP/TEMP environment variable only if dir
      is NULL, instead of using it regardless if it is set.
      We also want it to be safer than tempnam().
      If we return a filename, we are saying that the file did not exist
      at the time this function was called (and it wasn't a symlink pointing
      to a file that didn't exist, either).
      If dir is NULL this is a temp file in a public directory. In that
      case we create the file with permission 0600 before returning.

  Args: dir      -- The directory to create the name in
        prefix   -- Prefix of the name

 Result: Malloc'd string equal to new name is returned.  It must be free'd
	 by the caller.  Returns the string on success and NULL on failure.
  ----*/
char *
temp_nam(dir, prefix)
    char *dir, *prefix;
{
    struct stat buf;
    size_t      l, ll;
    char       *f, *name;

    if(!(name = (char *)malloc((unsigned int)NFILEN)))
        return((char *)NULL);

#ifndef __riscos
    if(!dir && (f = getenv("TMPDIR")) && !stat(f, &buf) &&
                         (buf.st_mode&S_IFMT) == S_IFDIR &&
			 !can_access(f, WRITE_ACCESS|EXECUTE_ACCESS)){
	strncpy(name, f, NFILEN-1);
	name[NFILEN-1] = '\0';
        goto done;
    }

    if(!dir && (f = getenv("TMP")) && !stat(f, &buf) &&
                         (buf.st_mode&S_IFMT) == S_IFDIR &&
			 !can_access(f, WRITE_ACCESS|EXECUTE_ACCESS)){
	strncpy(name, f, NFILEN-1);
	name[NFILEN-1] = '\0';
        goto done;
    }

    if(!dir && (f = getenv("TEMP")) && !stat(f, &buf) &&
                         (buf.st_mode&S_IFMT) == S_IFDIR &&
			 !can_access(f, WRITE_ACCESS|EXECUTE_ACCESS)){
	strncpy(name, f, NFILEN-1);
	name[NFILEN-1] = '\0';
        goto done;
    }

    if(dir && !stat(dir, &buf) &&
                         (buf.st_mode&S_IFMT) == S_IFDIR &&
	                 !can_access(dir, WRITE_ACCESS|EXECUTE_ACCESS)){
	strncpy(name, dir, NFILEN-1);
	name[NFILEN-1] = '\0';
        goto done;
    }

#ifndef P_tmpdir
#define	P_tmpdir	"/usr/tmp"
#endif
    if(!stat(P_tmpdir, &buf) &&
                         (buf.st_mode&S_IFMT) == S_IFDIR &&
			 !can_access(P_tmpdir, WRITE_ACCESS|EXECUTE_ACCESS)){
	strncpy(name, P_tmpdir, NFILEN-1);
	name[NFILEN-1] = '\0';
        goto done;
    }

    if(!stat("/tmp", &buf) &&
                         (buf.st_mode&S_IFMT) == S_IFDIR &&
			 !can_access("/tmp", WRITE_ACCESS|EXECUTE_ACCESS)){
	strncpy(name, "/tmp", NFILEN-1);
	name[NFILEN-1] = '\0';
        goto done;
    }

    free((void *)name);
    return((char *)NULL);

done:
    if(name[0] && *((f = &name[l=strlen(name)]) - 1) != '/' && l+1 < NFILEN){
	*f++ = '/';
	*f = '\0';
	l++;
    }

    if(prefix && (ll = strlen(prefix)) && l+ll < NFILEN){
	strcpy(f, prefix);
	f += ll;
	l += ll;
    }

    if(l+6 < NFILEN)
      strcpy(f, "XXXXXX");
    else{
	free((void *)name);
	return((char *)NULL);
    }

    return(was_nonexistent_tmp_name(name, 1, NULL));
#else
  {
    static int cnt=0;
    if (dir==NULL)
      sprintf(name, "<Wimp$ScrapDir>.%s%i-%d", prefix, cnt++, getpid());
    else
      sprintf(name, "%s.%s%i-%d", dir, prefix, cnt++, getpid());
  }
  return name;
#endif
}

/*----------------------------------------------------------------------

     Like temp_nam but create a unique name with an extension.

 Result: Malloc'd string equal to new name is returned.  It must be free'd
	 by the caller.  Returns the string on success and NULL on failure.
  ----*/
char *
temp_nam_ext(dir, prefix, ext)
    char *dir, *prefix, *ext;
{
    struct stat buf;
    size_t      l, ll;
    char       *f, *name;

    if(ext == NULL || *ext == '\0')
      return(temp_nam(dir, prefix));
    if(!(name = (char *)malloc((unsigned int)NFILEN)))
        return((char *)NULL);

#ifndef __riscos
    if(!dir && (f = getenv("TMPDIR")) && !stat(f, &buf) &&
                         (buf.st_mode&S_IFMT) == S_IFDIR &&
			 !can_access(f, WRITE_ACCESS|EXECUTE_ACCESS)){
	strncpy(name, f, NFILEN-1);
	name[NFILEN-1] = '\0';
        goto done;
    }

    if(!dir && (f = getenv("TMP")) && !stat(f, &buf) &&
                         (buf.st_mode&S_IFMT) == S_IFDIR &&
			 !can_access(f, WRITE_ACCESS|EXECUTE_ACCESS)){
	strncpy(name, f, NFILEN-1);
	name[NFILEN-1] = '\0';
        goto done;
    }

    if(!dir && (f = getenv("TEMP")) && !stat(f, &buf) &&
                         (buf.st_mode&S_IFMT) == S_IFDIR &&
			 !can_access(f, WRITE_ACCESS|EXECUTE_ACCESS)){
	strncpy(name, f, NFILEN-1);
	name[NFILEN-1] = '\0';
        goto done;
    }

    if(dir && !stat(dir, &buf) &&
                         (buf.st_mode&S_IFMT) == S_IFDIR &&
	                 !can_access(dir, WRITE_ACCESS|EXECUTE_ACCESS)){
	strncpy(name, dir, NFILEN-1);
	name[NFILEN-1] = '\0';
        goto done;
    }

#ifndef P_tmpdir
#define	P_tmpdir	"/usr/tmp"
#endif
    if(!stat(P_tmpdir, &buf) &&
                         (buf.st_mode&S_IFMT) == S_IFDIR &&
			 !can_access(P_tmpdir, WRITE_ACCESS|EXECUTE_ACCESS)){
	strncpy(name, P_tmpdir, NFILEN-1);
	name[NFILEN-1] = '\0';
        goto done;
    }

    if(!stat("/tmp", &buf) &&
                         (buf.st_mode&S_IFMT) == S_IFDIR &&
			 !can_access("/tmp", WRITE_ACCESS|EXECUTE_ACCESS)){
	strncpy(name, "/tmp", NFILEN-1);
	name[NFILEN-1] = '\0';
        goto done;
    }

    free((void *)name);
    return((char *)NULL);

done:
    if(name[0] && *((f = &name[l=strlen(name)]) - 1) != '/' && l+1 < NFILEN){
	*f++ = '/';
	*f = '\0';
	l++;
    }

    if(prefix && (ll = strlen(prefix)) && l+ll < NFILEN){
	strcpy(f, prefix);
	f += ll;
	l += ll;
    }

    if(l+6+strlen(ext)+1 < NFILEN)
      strcpy(f, "XXXXXX");
    else{
	free((void *)name);
	return((char *)NULL);
    }

    return(was_nonexistent_tmp_name(name, 1, ext));
#else
  {
    static int cnt=0;
    if (dir==NULL)
      sprintf(name, "<Wimp$ScrapDir>.%s%i-%d/%s", prefix, cnt++, getpid(), ext);
    else
      sprintf(name, "%s.%s%i-%d/%s", dir, prefix, cnt++, getpid(), ext);
  }
  return name;
#endif
}


/*
 * tmpname - return a temporary file name in the given buffer, the filename
 * is in the directory dir unless dir is NULL. The file did not exist at
 * the time of the temp_nam call, but was created by temp_nam.
 */
void
tmpname(dir, name)
char *dir;
char *name;
{
    char *t;

#ifndef __riscos
    if(t = temp_nam((dir && *dir) ? dir : NULL, "pico.")){
#else
    if(t = temp_nam((dir && *dir) ? dir : NULL, "pico")){
#endif
	strncpy(name, t, NFILEN-1);
	name[NFILEN-1] = '\0';
	free((void *)t);
    }
    else {
	emlwrite("Unable to construct temp file name", NULL);
	name[0] = '\0';
    }
}


/*
 * Take a file name, and from it
 * fabricate a buffer name. This routine knows
 * about the syntax of file names on the target system.
 * I suppose that this information could be put in
 * a better place than a line of code.
 */
void
makename(bname, fname)
char    bname[];
char    fname[];
{
    register char   *cp1;
    register char   *cp2;

    cp1 = &fname[0];
    while (*cp1 != 0)
      ++cp1;

#ifdef __riscos
    while (cp1!=&fname[0] && cp1[-1]!='/')
      --cp1;
#else
    while (cp1!=&fname[0] && cp1[-1]!='.')
      --cp1;
#endif

    cp2 = &bname[0];
    while (cp2!=&bname[NBUFN-1] && *cp1!=0 && *cp1!=';')
      *cp2++ = *cp1++;

    *cp2 = 0;
}


/*
 * copy - copy contents of file 'a' into a file named 'b'.  Return error
 *        if either isn't accessible or is a directory
 */
copy(a, b)
char *a, *b;
{
#ifndef __riscos
    int    in, out, n, rv = 0;
    char   *cb;
    struct stat tsb, fsb;
    extern int  errno;

    if(stat(a, &fsb) < 0){		/* get source file info */
	emlwrite("Can't Copy: %s", errstr(errno));
	return(-1);
    }

    if(!(fsb.st_mode&S_IREAD)){		/* can we read it? */
	emlwrite("\007Read permission denied: %s", a);
	return(-1);
    }

    if((fsb.st_mode&S_IFMT) == S_IFDIR){ /* is it a directory? */
	emlwrite("\007Can't copy: %s is a directory", a);
	return(-1);
    }

    if(stat(b, &tsb) < 0){		/* get dest file's mode */
	switch(errno){
	  case ENOENT:
	    break;			/* these are OK */
	  default:
	    emlwrite("\007Can't Copy: %s", errstr(errno));
	    return(-1);
	}
    }
    else{
	if(!(tsb.st_mode&S_IWRITE)){	/* can we write it? */
	    emlwrite("\007Write permission denied: %s", b);
	    return(-1);
	}

	if((tsb.st_mode&S_IFMT) == S_IFDIR){	/* is it directory? */
	    emlwrite("\007Can't copy: %s is a directory", b);
	    return(-1);
	}

	if(fsb.st_dev == tsb.st_dev && fsb.st_ino == tsb.st_ino){
	    emlwrite("\007Identical files.  File not copied", NULL);
	    return(-1);
	}
    }

    if((in = open(a, O_RDONLY)) < 0){
	emlwrite("Copy Failed: %s", errstr(errno));
	return(-1);
    }

    if((out=creat(b, fsb.st_mode&0xfff)) < 0){
	emlwrite("Can't Copy: %s", errstr(errno));
	close(in);
	return(-1);
    }

    if((cb = (char *)malloc(NLINE*sizeof(char))) == NULL){
	emlwrite("Can't allocate space for copy buffer!", NULL);
	close(in);
	close(out);
	return(-1);
    }

    while(1){				/* do the copy */
	if((n = read(in, cb, NLINE)) < 0){
	    emlwrite("Can't Read Copy: %s", errstr(errno));
	    rv = -1;
	    break;			/* get out now */
	}

	if(n == 0)			/* done! */
	  break;

	if(write(out, cb, n) != n){
	    emlwrite("Can't Write Copy: %s", errstr(errno));
	    rv = -1;
	    break;
	}
    }

    free(cb);
    close(in);
    close(out);
    return(rv);
#else
  int regs[11];
  _kernel_oserror *err;
  regs[0]=26; /* copy objects */
  regs[1]=(int)a;  /* source */
  regs[2]=(int)b;  /* destination */
  regs[3]=1+2+(1<<9); /* Recurse, Force, Access as source */
  err=_swix(OS_FSControl,_INR(0,3),26,a,b,1+2+(1<<9));

    if (err!=NULL){		/* get source file info */
	emlwrite("Can't copy: %s", err->errmess);
	return(-1);
    }
    return(0);
#endif
}


/*
 * Open a file for writing. Return TRUE if all is well, and FALSE on error
 * (cannot create).
 */
ffwopen(fn, readonly)
char    *fn;
int	 readonly;
{
    extern FIOINFO g_pico_fio;
#ifdef __riscos
    FILE *ffp;

/* JRF: Note: We ignore the readonly bit */
    if ((ffp=fopen(fn, "w")) == NULL) {
        emlwrite("Cannot open file for writing", NULL);
        return (FIOERR);
    }
    g_pico_fio.fp=ffp;

    return (FIOSUC);
#else
    int		 fd;
#ifndef	MODE_READONLY
#define	MODE_READONLY	(0600)
#endif

    /*
     * Call open() by hand since we don't want O_TRUNC -- it'll
     * screw over-quota users.  The strategy is to overwrite the
     * existing file's data and call ftruncate before close to lop
     * off bytes
     */

    g_pico_fio.flags = FIOINFO_WRITE;
    g_pico_fio.name  = fn;
    if((fd = open(fn, O_CREAT|O_WRONLY, readonly ? MODE_READONLY : 0666)) >= 0
       && (g_pico_fio.fp = fdopen(fd, "w")) != NULL
       && fseek(g_pico_fio.fp, 0L, 0) == 0)
      return (FIOSUC);

    emlwrite("Cannot open file for writing: %s", errstr(errno));
    return (FIOERR);
#endif
}


/*
 * Close a file. Should look at the status in all systems.
 */
ffclose()
{
    extern FIOINFO g_pico_fio;
#ifdef __riscos
    if (fclose(g_pico_fio.fp) != FALSE) {
        emlwrite("Error closing file", NULL);
        return(FIOERR);
    }

    return(FIOSUC);
#else

    errno = 0;
    if((g_pico_fio.flags & FIOINFO_WRITE)
       && (fflush(g_pico_fio.fp) == EOF
	   || ftruncate(fileno(g_pico_fio.fp),
			(off_t) ftell(g_pico_fio.fp)) < 0)){
	emlwrite("\007Error preparing to close file: %s", errstr(errno));
	sleep(5);
    }

    if (fclose(g_pico_fio.fp) == EOF) {
        emlwrite("\007Error closing file: %s", errstr(errno));
        return(FIOERR);
    }
#endif

    return(FIOSUC);
}



#define	EXTEND_BLOCK	1024


/*
 * ffelbowroom - make sure the destination's got enough room to receive
 *		 what we're about to write...
 */
ffelbowroom()
{
#ifndef __riscos
    register LINE *lp;
    register long  n;
    int		   x;
    char	   s[EXTEND_BLOCK], *errstring = NULL;
    struct stat	   fsbuf;
    extern FIOINFO g_pico_fio;

    /* Figure out how much room do we need */
    /* first, what's total */
    for(n=0L, lp=lforw(curbp->b_linep); lp != curbp->b_linep; lp=lforw(lp))
      n += (llength(lp) + 1);

    errno = 0;			/* make sure previous error are cleared */

    if(fstat(fileno(g_pico_fio.fp), &fsbuf) == 0){
	n -= fsbuf.st_size;

	if(n > 0L){			/* must be growing file, extend it */
	    memset(s, 'U', EXTEND_BLOCK);
	    if(fseek(g_pico_fio.fp, fsbuf.st_size, 0) == 0){
		for( ; n > 0L; n -= EXTEND_BLOCK){
		    x = (n < EXTEND_BLOCK) ? (int) n : EXTEND_BLOCK;
		    if(fwrite(s, x * sizeof(char), 1, g_pico_fio.fp) != 1){
			errstring = errstr(errno);
			break;
		    }
		}

		if(!errstring
		   && (fflush(g_pico_fio.fp) == EOF
		       || fsync(fileno(g_pico_fio.fp)) < 0))
		  errstring = errstr(errno);

		if(errstring)			/* clean up */
		  (void) ftruncate(fileno(g_pico_fio.fp), fsbuf.st_size);
		else if(fseek(g_pico_fio.fp, 0L, 0) != 0)
		  errstring = errstr(errno);
	    }
	    else
	      errstring = errstr(errno);
	}
    }
    else
      errstring = errstr(errno);

    if(errstring){
	sprintf(s, "Error writing to %s: %s", g_pico_fio.name, errstring);
	emlwrite(s, NULL);
	(void) fclose(g_pico_fio.fp);
	return(FALSE);
    }
#else
    /* No implementation room for RISC OS */
#endif

    return(TRUE);
}


/*
 * P_open - run the given command in a sub-shell returning a file pointer
 *	    from which to read the output
 *
 * note:
 *	For OS's other than unix, you will have to rewrite this function.
 *	Hopefully it'll be easy to exec the command into a temporary file,
 *	and return a file pointer to that opened file or something.
 */
FILE *P_open(s)
char *s;
{
#ifndef __riscos
    return(popen(s, "r"));
#else
    printf("P_open not implemented\n");
    return NULL;
#endif
}



/*
 * P_close - close the given descriptor
 *
 */
void
P_close(fp)
FILE *fp;
{
#ifndef __riscos
    pclose(fp);
#else
    printf("P_close not implemented\n");
#endif
}



/*
 * worthit - generic sort of test to roughly gage usefulness of using
 *           optimized scrolling.
 *
 * note:
 *	returns the line on the screen, l, that the dot is currently on
 */
worthit(l)
int *l;
{
    int i;			/* l is current line */
    unsigned below;		/* below is avg # of ch/line under . */

    *l = doton(&i, &below);
    below = (i > 0) ? below/(unsigned)i : 0;

    return(below > 3);
}



/*
 * pico_new_mail - just checks mtime and atime of mail file and notifies user
 *	           if it's possible that they have new mail.
 */
pico_new_mail()
{
    int ret = 0;
    static time_t lastchk = 0;
    struct stat sbuf;
    char   inbox[256], *p;

#ifndef __riscos
    if(p = (char *)getenv("MAIL"))
      /* fix unsafe sprintf - noticed by petter wahlman <petter@bluezone.no> */
      sprintf(inbox, "%.*s", sizeof(inbox)-1, p);
    else
      sprintf(inbox,"%.*s/%.*s", sizeof(inbox)/3, MAILDIR, sizeof(inbox)/3,
	      (char *) getlogin());

    if(stat(inbox, &sbuf) == 0){
	ret = sbuf.st_atime <= sbuf.st_mtime &&
	  (lastchk < sbuf.st_mtime && lastchk < sbuf.st_atime);
	lastchk = sbuf.st_mtime;
	return(ret);
    }
    else
#endif
      return(ret);
}



/*
 * time_to_check - checks the current time against the last time called
 *                 and returns true if the elapsed time is > below.
 *                 Newmail won't necessarily check, but we want to give it
 *                 a chance to check or do a keepalive.
 */
time_to_check()
{
    static time_t lasttime = 0L;

    if(!timeo)
      return(FALSE);

    if(time((time_t *) 0) - lasttime > (Pmaster ? (time_t)(FUDGE-10) : timeo)){
	lasttime = time((time_t *) 0);
	return(TRUE);
    }
    else
      return(FALSE);
}


/*
 * sstrcasecmp - compare two pointers to strings case independently
 */
int
sstrcasecmp(s1, s2)
    const QSType *s1, *s2;
{
    return((*pcollator)(*(char **)s1, *(char **)s2));
}


/*--------------------------------------------------
     A case insensitive strcmp()

   Args: o, r -- The two strings to compare

 Result: integer indicating which is greater
  ---*/
int
strucmp(o, r)
    register char *o, *r;
{
    if(o == NULL){
	if(r == NULL)
	  return 0;
	else
	  return -1;
    }
    else if(r == NULL)
      return 1;

    while(*o && *r
	  && ((isupper((unsigned char)(*o))
				  ? (unsigned char)tolower((unsigned char)(*o))
				  : (unsigned char)(*o))
	     == (isupper((unsigned char)(*r))
				  ? (unsigned char)tolower((unsigned char)(*r))
				  : (unsigned char)(*r)))){
	o++;
	r++;
    }

    return((isupper((unsigned char)(*o))
				? tolower((unsigned char)(*o))
				: (int)(unsigned char)(*o))
	   - (isupper((unsigned char)(*r))
			        ? tolower((unsigned char)(*r))
				: (int)(unsigned char)(*r)));
}


/*----------------------------------------------------------------------
     A case insensitive strncmp()

   Args: o, r -- The two strings to compare
         n    -- length to stop comparing strings at

 Result: integer indicating which is greater

  ----*/
int
struncmp(o, r, n)
    register char *o,
		  *r;
    register int   n;
{
    if(n < 1)
      return 0;

    if(o == NULL){
	if(r == NULL)
	  return 0;
	else
	  return -1;
    }
    else if(r == NULL)
      return 1;

    n--;
    while(n && *o && *r
	  && ((isupper((unsigned char)(*o))
				  ? (unsigned char)tolower((unsigned char)(*o))
				  : (unsigned char)(*o))
	     == (isupper((unsigned char)(*r))
				  ? (unsigned char)tolower((unsigned char)(*r))
				  : (unsigned char)(*r)))){
	o++;
	r++;
	n--;
    }

    return((isupper((unsigned char)(*o))
				? tolower((unsigned char)(*o))
				: (int)(unsigned char)(*o))
	   - (isupper((unsigned char)(*r))
			        ? tolower((unsigned char)(*r))
				: (int)(unsigned char)(*r)));
}


/*
 * chkptinit -- initialize anything we need to support composer
 *		checkpointing
 */
void
chkptinit(file, n)
    char *file;
    int   n;
{
#ifdef __riscos
    unsigned pid;
    char    *chp;

    if(!file[0]){
	long gmode_save = gmode;

	if(gmode&MDCURDIR)
	  gmode &= ~MDCURDIR;  /* so fixpath will use home dir */

	strcpy(file, "#picoXXXXX#");
	fixpath(file, NLINE);
	gmode = gmode_save;
    }
    else{
	int l = strlen(file);

	if(file[l-1] != '/'){
	    file[l++] = '/';
	    file[l]   = '\0';
	}

	strcpy(file + l, "#picoXXXXX#");
    }

    pid = (unsigned)getpid();
    for(chp = file+strlen(file) - 2; *chp == 'X'; chp--){
	*chp = (pid % 10) + '0';
	pid /= 10;
    }
#else
    unsigned pid;
    char    *chp;

    if(!file[0]){
	long gmode_save = gmode;

	if(gmode&MDCURDIR)
	  gmode &= ~MDCURDIR;  /* so fixpath will use home dir */

	strcpy(file, "picoXXXXX");
	fixpath(file, NLINE);
	gmode = gmode_save;
    }
    else{
	int l = strlen(file);

/* Gerph's change to work with arc dirs (Plus #'s removed from names)*/
	if(file[l-1] != '.'){
	    file[l++] = '.';
	    file[l]   = '\0';
	}

	strcpy(file + l, "picoXXXXX");
    }

    pid = (unsigned)getpid();
    for(chp = file+strlen(file) - 2; *chp == 'X'; chp--){
	*chp = (pid % 10) + '0';
	pid /= 10;
    }
#endif

    remove(file);
}

void
set_collation(collation, ctype)
    int collation;
    int ctype;
{
    extern int collator();  /* strcoll isn't declared on all systems */
#ifdef LC_COLLATE
    char *status = NULL;
#endif

    pcollator = strucmp;

#ifdef LC_COLLATE
  if(collation){
    /*
     * This may not have the desired effect, if collator is not
     * defined to be strcoll in os.h and strcmp and friends
     * don't know about locales. If your system does have strcoll
     * but we haven't defined collator to be strcoll in os.h, let us know.
     */
    status = setlocale(LC_COLLATE, "");

    /*
     * If there is an error or if the locale is the "C" locale, then we
     * don't want to use strcoll because in the default "C" locale strcoll
     * uses strcmp ordering and we want strucmp ordering.
     *
     * The problem with this is that setlocale returns a string which is
     * not equal to "C" on some systems even when the locale is "C", so we
     * can't really tell on those systems. On some systems like that, we
     * may end up with a strcmp-style collation instead of a strucmp-style.
     * We recommend that the users of those systems explicitly set
     * LC_COLLATE in their environment.
     */
    if(status && !(status[0] == 'C' && status[1] == '\0'))
      pcollator = collator;
  }
#endif
#ifdef LC_CTYPE
  if(ctype){
    (void)setlocale(LC_CTYPE, "");
  }
#endif
}


#ifdef	RESIZING
/*
 * winch_handler - handle window change signal
 */
SigType winch_handler SIG_PROTO((int sig))
{
    signal(SIGWINCH, winch_handler);
    ttresize();
    if(Pmaster && Pmaster->winch_cleanup && Pmaster->arm_winch_cleanup)
      (*Pmaster->winch_cleanup)();
}
#endif	/* RESIZING */


#ifdef	SIGCHLD
/*
 * child_handler - handle child status change signal
 */
SigType child_handler SIG_PROTO ((int sig))
{
    pico_child_done = 1;
    if(pico_child_jmp_ok){
#ifdef	sco
	/*
	 * Villy Kruse <vek@twincom.nl> says:
	 * This probably only affects older unix systems with a "broken" sleep
	 * function such as AT&T System V rel 3.2 and systems derived from
	 * that version. The problem is that the sleep function on these
	 * versions of unix will set up a signal handler for SIGALRM which
	 * will longjmp into the sleep function when the alarm goes off.
	 * This gives problems if another signal handler longjmps out of the
	 * sleep function, thus making the stack frame for the signal function
	 * invalid, and when the ALARM handler later longjmps back into the
	 * sleep function it does no longer have a valid stack frame.
	 * My sugested fix is to cancel the pending alarm in the SIGCHLD
	 * handler before longjmp'ing. This shouldn't hurt as there
	 * shouldn't be any ALARM pending at this point except possibly from
	 * the sleep call.
	 *
	 *
	 * [Even though it shouldn't hurt, we'll make it sco-specific for now.]
	 *
	 *
	 * The sleep call might have set up a signal handler which would
	 * longjmp back into the sleep code, and that would cause a crash.
	 */
	signal(SIGALRM, SIG_IGN);	/* Cancel signal handeler */
	alarm(0);			/* might longjmp back into sleep */
#endif
	longjmp(pico_child_state, 1);
    }
}
#endif	/* SIGCHLD */


#ifdef	POSIX_SIGNALS
/*----------------------------------------------------------------------
   Reset signals after critical imap code
 ----*/
SigType
(*posix_signal(sig_num, action))()
    int	    sig_num;
    SigType (*action)();
{
    struct sigaction new_action, old_action;

    memset((void *)&new_action, 0, sizeof(struct sigaction));
    sigemptyset (&new_action.sa_mask);
    new_action.sa_handler = action;
#ifdef	SA_RESTART
    new_action.sa_flags = SA_RESTART;
#else
    new_action.sa_flags = 0;
#endif
    sigaction(sig_num, &new_action, &old_action);
    return(old_action.sa_handler);
}

int
posix_sigunblock(mask)
    int mask;
{
    sigset_t sig_mask;

    sigemptyset(&sig_mask);
    sigaddset(&sig_mask, mask);
    return(sigprocmask(SIG_UNBLOCK, &sig_mask, NULL));
}
#endif /* POSIX_SIGNALS */


#if	defined(sv3) || defined(ct)
/* Placed by rll to handle the rename function not found in AT&T */
rename(oldname, newname)
    char *oldname;
    char *newname;
{
    int rtn;

    if ((rtn = link(oldname, newname)) != 0) {
	perror("Was not able to rename file.");
	return(rtn);
    }

    if ((rtn = unlink(oldname)) != 0)
      perror("Was not able to unlink file.");

    return(rtn);
}
#endif


#ifdef	MOUSE

/*
 * init_mouse - check for xterm and initialize mouse tracking if present...
 */
init_mouse()
{
    if(mexist)
      return(TRUE);

    if(getenv("DISPLAY")){
	mouseon();
        kpinsert("\033[M", KEY_XTERM_MOUSE, 1);
	return(mexist = TRUE);
    }
    else
      return(FALSE);
}


/*
 * end_mouse - clear xterm mouse tracking if present...
 */
void
end_mouse()
{
    if(mexist){
	mexist = 0;			/* just see if it exists here. */
	mouseoff();
    }
}


/*
 * mouseexist - function to let outsiders know if mouse is turned on
 *              or not.
 */
mouseexist()
{
    return(mexist);
}


/*
 * mouseon - call made available for programs calling pico to turn ON the
 *           mouse cursor.
 */
void
mouseon()
{
    fputs(XTERM_MOUSE_ON, stdout);
}


/*
 * mouseon - call made available for programs calling pico to turn OFF the
 *           mouse cursor.
 */
void
mouseoff()
{
    fputs(XTERM_MOUSE_OFF, stdout);
}


/*
 * checkmouse - look for mouse events in key menu and return
 *              appropriate value.
 */
int
checkmouse(ch, down, mcol, mrow)
unsigned *ch;
int	  down, mcol, mrow;
{
    static int oindex;
    int i = 0, rv = 0;
    MENUITEM *mp;

    if(!mexist || mcol < 0 || mrow < 0)
      return(FALSE);

    if(down)			/* button down */
      oindex = -1;

    for(mp = mfunc; mp; mp = mp->next)
      if(mp->action && M_ACTIVE(mrow, mcol, mp))
	break;

    if(mp){
	unsigned long r;

	r = (*mp->action)(down ? M_EVENT_DOWN : M_EVENT_UP,
			  mrow, mcol, M_BUTTON_LEFT, 0);
	if(r & 0xffff){
	    *ch = (unsigned)((r>>16)&0xffff);
	    rv  = TRUE;
	}
    }
    else{
	while(1){			/* see if we understand event */
	    if(i >= 12){
		i = -1;
		break;
	    }

	    if(M_ACTIVE(mrow, mcol, &menuitems[i]))
	      break;

	    i++;
	}

	if(down){			/* button down */
	    oindex = i;			/* remember where */
	    if(i != -1
	       && menuitems[i].label_hiliter != NULL
	       && menuitems[i].val != mnoop)  /* invert label */
	      (*menuitems[i].label_hiliter)(1, &menuitems[i]);
	}
	else{				/* button up */
	    if(oindex != -1){
		if(i == oindex){
		    *ch = menuitems[i].val;
		    rv = TRUE;
		}
	    }
	}
    }

    /* restore label */
    if(!down
       && oindex != -1
       && menuitems[oindex].label_hiliter != NULL
       && menuitems[oindex].val != mnoop)
      (*menuitems[oindex].label_hiliter)(0, &menuitems[oindex]);

    return(rv);
}


/*
 * invert_label - highlight the label of the given menu item.
 */
void
invert_label(state, m)
int state;
MENUITEM *m;
{
    unsigned i, j;
    int   col_offset, savettrow, savettcol;
    char *lp;

    get_cursor(&savettrow, &savettcol);

    /*
     * Leave the command name bold
     */
    col_offset = (state || !(lp=strchr(m->label, ' '))) ? 0 : (lp - m->label);
    movecursor((int)m->tl.r, (int)m->tl.c + col_offset);
    flip_inv(state);

    for(i = m->tl.r; i <= m->br.r; i++)
      for(j = m->tl.c + col_offset; j <= m->br.c; j++)
	if(i == m->lbl.r && j == m->lbl.c + col_offset && m->label){
	    lp = m->label + col_offset;		/* show label?? */
	    while(*lp && j++ < m->br.c)
	      putc(*lp++, stdout);

	    continue;
	}
	else
	  putc(' ', stdout);

    if(state)
      flip_inv(FALSE);

    movecursor(savettrow, savettcol);
}
#endif	/* MOUSE */


/*
 * Program:	spell.c
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
 * 1989-2002 by the University of Washington.
 *
 * The full text of our legal notices is contained in the file called
 * CPYRIGHT, included with this distribution.
 *
 */

#ifdef	SPELLER

int  movetoword PROTO((char *));

#define  NSHLINES  12

static char *spellhelp[] = {
  "Spell Check Help",
  " ",
  "\tThe spell checker examines all words in the text.  It then",
  "\toffers each misspelled word for correction while simultaneously",
  "\thighlighting it in the text.  To leave a word unchanged simply",
  "~\thit ~R~e~t~u~r~n at the edit prompt.  If a word has been corrected,",
  "\teach occurrence of the incorrect word is offered for replacement.",
  " ",
  "~\tSpell checking can be cancelled at any time by typing ~^~C (~F~3)",
  "\tafter exiting help.",
  " ",
  "End of Spell Check Help",
  " ",
  NULL
};


static char *pinespellhelp[] = {
  "Spell Check Help",
  " ",
  "\tThe spell checker examines all words in the text.  It then",
  "\toffers each misspelled word for correction while simultaneously",
  "\thighlighting it in the text.  To leave a word unchanged simply",
  "\thit Return at the edit prompt.  If a word has been corrected,",
  "\teach occurrence of the incorrect word is offered for replacement.",
  " ",
  "\tSpell checking can be cancelled at any time by typing ^C (F3)",
  "\tafter exiting help.",
  " ",
  "End of Spell Check Help",
  " ",
  NULL
};



/*
 * spell() - check for potentially missspelled words and offer them for
 *           correction
 */
spell(f, n)
    int f, n;
{
    int    status, next, ret;
    FILE   *p;
    char   *b, *sp, *fn;
    char   wb[NLINE], cb[NLINE];
    FILE   *P_open();

    setimark(0, 1);
    emlwrite("Checking spelling...", NULL); 	/* greetings */

    if(alt_speller)
      return(alt_editor(1, 0));			/* f == 1 means fork speller */

    if((fn = writetmp(0, NULL)) == NULL){
	emlwrite("Can't write temp file for spell checker", NULL);
	return(-1);
    }

#ifdef __riscos
    if((sp = (char *)getenv("Pico$Spell")) == NULL)
#else
    if((sp = (char *)getenv("SPELL")) == NULL)
#endif
      sp = SPELLER;

    if(fexist(sp, "x", (off_t *)NULL) != FIOSUC){
        emlwrite("\007Spell-checking file \"%s\" not found", sp);
	return(-1);
    }

#ifdef __riscos
  {
    char tempfile[NLINE];
    tmpname(NULL,tempfile);

    sprintf(cb, "%s < %s > %s", sp, fn, tempfile);	/* pre-use buffer! */
    if (system(cb)<0)
    {
	remove(fn);
	emlwrite("Can't launch spell checker", NULL);
	return(-1);
    }

    if((p = fopen(tempfile,"r")) == NULL){ 		/* read output from command */
	remove(fn);
	remove(tempfile);
	emlwrite("Can't read spell checker output", NULL);
	return(-1);
    }
#else
    sprintf(cb, "( %s ) < %s", sp, fn);		/* pre-use buffer! */
    if((p = P_open(cb)) == NULL){ 		/* read output from command */
	unlink(fn);
	emlwrite("Can't fork spell checker", NULL);
	return(-1);
    }
#endif

    ret = 1;
    while(fgets(wb, NLINE, p) != NULL && ret){
	if((b = (char *)strchr(wb,'\n')) != NULL)
	  *b = '\0';
	strcpy(cb, wb);

	gotobob(0, 1);

	status = TRUE;
	next = 1;

	while(status){
	    if(next++)
	      if(movetoword(wb) != TRUE)
		break;

	    update();
	    (*term.t_rev)(1);
	    pputs(wb, 1);			/* highlight word */
	    (*term.t_rev)(0);

	    if(strcmp(cb, wb)){
		char prompt[2*NLINE + 32];
		sprintf(prompt, "Replace \"%s\" with \"%s\"", wb, cb);
		status=mlyesno(prompt, TRUE);
	    }
	    else
	      status=mlreplyd("Edit a replacement: ", cb, NLINE, QDEFLT, NULL);


	    curwp->w_flag |= WFMOVE;		/* put cursor back */
	    sgarbk = 0;				/* fake no-keymenu-change! */
	    update();
	    pputs(wb, 0);			/* un-highlight */

	    switch(status){
	      case TRUE:
		chword(wb, cb);			/* correct word    */
	      case FALSE:
		update();			/* place cursor */
		break;
	      case ABORT:
		emlwrite("Spell Checking Cancelled", NULL);
		ret = FALSE;
		status = FALSE;
		break;
	      case HELPCH:
		if(Pmaster){
		    VARS_TO_SAVE *saved_state;

		    saved_state = save_pico_state();
		    (*Pmaster->helper)(pinespellhelp,
				       "Help with Spelling Checker", 1);
		    if(saved_state){
			restore_pico_state(saved_state);
			free_pico_state(saved_state);
		    }
		}
		else
		  pico_help(spellhelp, "Help with Spelling Checker", 1);
	      case (CTRL|'L'):
		next = 0;			/* don't get next word */
		sgarbf = TRUE;			/* repaint full screen */
		update();
		status = TRUE;
		continue;
	      default:
		emlwrite("Huh?", NULL);		/* shouldn't get here, but.. */
		status = TRUE;
		sleep(1);
		break;
	    }
	    forwword(0, 1);			/* goto next word */
	}
    }
#ifdef __riscos
    fclose(p);					/* clean up */
#else
    P_close(p);					/* clean up */
#endif
    remove(fn);
#ifdef __riscos
    remove(tempfile);
  }
#endif
    swapimark(0, 1);
    curwp->w_flag |= WFHARD|WFMODE;
    sgarbk = TRUE;

    if(ret)
      emlwrite("Done checking spelling", NULL);
    return(ret);
}



/*
 * movetoword() - move to the first occurance of the word w
 *
 *	returns:
 *		TRUE upon success
 *		FALSE otherwise
 */
movetoword(w)
  char *w;
{
    int      i;
    int      ret  = FALSE;
    int	     olddoto;
    LINE     *olddotp;
    register int   off;				/* curwp offset */
    register LINE *lp;				/* curwp line   */

    olddoto = curwp->w_doto;			/* save where we are */
    olddotp = curwp->w_dotp;

    curwp->w_bufp->b_mode |= MDEXACT;		/* case sensitive */
    while(forscan(&i, w, NULL, 0, 1) == TRUE){
	if(i)
	  break;				/* wrap NOT allowed! */

	lp  = curwp->w_dotp;			/* for convenience */
	off = curwp->w_doto;

	/*
	 * We want to minimize the number of substrings that we report
	 * as matching a misspelled word...
	 */
	if(off == 0 || !isalpha((unsigned char)lgetc(lp, off - 1).c)){
	    off += strlen(w);
	    if((!isalpha((unsigned char)lgetc(lp, off).c) || off == llength(lp))
	       && lgetc(lp, 0).c != '>'){
		ret = TRUE;
		break;
	    }
	}

	forwchar(0, 1);				/* move on... */

    }
    curwp->w_bufp->b_mode ^= MDEXACT;		/* case insensitive */

    if(ret == FALSE){
	curwp->w_dotp = olddotp;
	curwp->w_doto = olddoto;
    }
    else
      curwp->w_flag |= WFHARD;

    return(ret);
}
#endif	/* SPELLER */

#ifdef __riscos
int cached_read=-1;
#endif


/*
 *  Check whether or not a character is ready to be read, or time out.
 *  This version uses a select call.
 *
 *   Args: time_out -- number of seconds before it will time out.
 *
 * Result: NO_OP_IDLE:	  timed out before any chars ready, time_out > 25
 *         NO_OP_COMMAND: timed out before any chars ready, time_out <= 25
 *         READ_INTR:	  read was interrupted
 *         READY_TO_READ: input is available
 *         BAIL_OUT:	  reading input failed, time to bail gracefully
 *         PANIC_NOW:	  system call error, die now
 */
int
input_ready(time_out)
     int time_out;
{
#ifdef __riscos
     int start=0;

     if (cached_read != -1)
       return READY_TO_READ;

     start=clock();

     /* printf("input_ready: timeout = %i\n", time_out); */

     do
     {
       int type;
       _swix(OS_UpCall,_INR(0,1),6,0);
       if (_swix(OS_Byte,_INR(0,2)|_OUTR(1,2),129,0,0,&cached_read,&type))
         return PANIC_NOW;
       if (type==0)
         return(READY_TO_READ);
       if (type==27)
       {
         cached_read=27;
         return READ_INTR;
       }
       cached_read=-1;
     }
     while ((clock()-start) < time_out*CLOCKS_PER_SEC);

     if (time_out>IDLE_TIMEOUT)
       return NO_OP_COMMAND;
     else
       return NO_OP_IDLE;
#else
     struct timeval tmo;
     fd_set         readfds, errfds;
     int	    res;

     fflush(stdout);

     if(time_out > 0){
         /* Check to see if there are bytes to read with a timeout */
	 FD_ZERO(&readfds);
	 FD_ZERO(&errfds);
	 FD_SET(STDIN_FD, &readfds);
	 FD_SET(STDIN_FD, &errfds);
	 tmo.tv_sec  = time_out;
         tmo.tv_usec = 0;
         res = select(STDIN_FD+1, &readfds, 0, &errfds, &tmo);
         if(res < 0){
             if(errno == EINTR || errno == EAGAIN)
               return(READ_INTR);

	     return(BAIL_OUT);
         }

         if(res == 0){ /* the select timed out */
	     if(getppid() == 1){
		 /* Parent is init! */
	         return(BAIL_OUT);
	     }

	     /*
	      * "15" is the minimum allowed mail check interval.
	      * Anything less, and we're being told to cycle thru
	      * the command loop because some task is pending...
	      */
             return(time_out < IDLE_TIMEOUT ? NO_OP_COMMAND : NO_OP_IDLE);
         }
     }

     time_of_last_input = time((time_t *) 0);
     return(READY_TO_READ);
#endif
}


/*
 * Read one character from STDIN.
 *
 * Result:           -- the single character read
 *         READ_INTR -- read was interrupted
 *         BAIL_OUT  -- read error of some sort
 */
int
read_one_char()
{
#ifdef __riscos
      int	    res;
      int            c;
      _kernel_oserror *err;

      if (cached_read!=-1)
      {
        c=cached_read;
        cached_read=-1;
      }
      else
      {

        err=_swix(OS_ReadC,_OUT(0),&c);

        if (err)
          return BAIL_OUT;
      }

      /* printf("read_one_char: %i\n", c); */

      switch (c)
      {
        case 0x81: c=F1;                 break;
        case 0x82: c=F2;                 break;
        case 0x83: c=F3;                 break;
        case 0x84: c=F4;                 break;
        case 0x85: c=F5;                 break;
        case 0x86: c=F6;                 break;
        case 0x87: c=F7;                 break;
        case 0x88: c=F8;                 break;
        case 0x89: c=F9;                 break;
        case 0xCA: c=F10;                break;
        case 0xCB: c=F11;                break;
        case 0xCC: c=F12;                break;
        case 0x8b: c=KEY_END;            break;
        case 0x8c: c=KEY_LEFT;           break;
        case 0x8d: c=KEY_RIGHT;          break;
        case 0x8e: c=KEY_DOWN;           break;
        case 0x8f: c=KEY_UP;             break;
        case 0x9e: c=KEY_PGDN;           break;
        case 0x9f: c=KEY_PGUP;           break;
        case 30:   c=KEY_HOME;           break;
        case 8:    c=0x7f;               break;
      }
      /* printf("  %i\n", c); */
#else
     int	    res;
     unsigned char  c;
     res = read(STDIN_FD, &c, 1);

     if(res <= 0){
	 /*
	  * Error reading from terminal!
	  * The only acceptable failure is being interrupted.  If so,
	  * return a value indicating such...
	  */
	 if(res < 0 && errno == EINTR)
	   return(READ_INTR);
	 else
	   return(BAIL_OUT);
     }

#endif
     return((int)c);
}


#ifndef __riscos
/*
 * TTY setup routines. These are the TERMIOS-style (POSIX) routines.
 */

static struct termios _raw_tty, _original_tty;

/*
 * current raw state
 */
static short _inraw = 0;


/*
 *  Set up the tty driver
 *
 * Args: state -- which state to put it in. 1 means go into raw (cbreak),
 *                0 out of raw.
 *
 * Result: returns 0 if successful and -1 if not.
 */
int
Raw(state)
int state;
{
    /** state is either ON or OFF, as indicated by call **/
    /* Check return code only on first call. If it fails we're done for and
       if it goes OK the others will probably go OK too. */

    if(state == 0 && _inraw){
        /*----- restore state to original -----*/
	if(tcsetattr(STDIN_FD, TCSADRAIN, &_original_tty) < 0)
	  return -1;

        _inraw = 0;
    }
    else if(state == 1 && ! _inraw){
        /*----- Go into raw mode (cbreak actually) ----*/
	if(tcgetattr(STDIN_FD, &_original_tty) < 0)
	  return -1;

	tcgetattr(STDIN_FD, &_raw_tty);
	_raw_tty.c_lflag &= ~(ICANON | ECHO | IEXTEN); /* noecho raw mode    */
 	_raw_tty.c_lflag &= ~ISIG;		/* disable signals           */
 	_raw_tty.c_iflag &= ~ICRNL;		/* turn off CR->NL on input  */
 	_raw_tty.c_oflag &= ~ONLCR;		/* turn off NL->CR on output */

	_raw_tty.c_cc[VMIN]  = '\01';		/* min # of chars to queue  */
	_raw_tty.c_cc[VTIME] = '\0';		/* min time to wait for input*/
	_raw_tty.c_cc[VINTR] = ctrl('C');	/* make it our special char */
	_raw_tty.c_cc[VQUIT] = 0;
	_raw_tty.c_cc[VSUSP] = 0;
	tcsetattr(STDIN_FD, TCSADRAIN, &_raw_tty);
	_inraw = 1;
    }

    return(0);
}


/*
 *  Set up the tty driver to use XON/XOFF flow control
 *
 * Args: state -- True to make sure XON/XOFF turned on, FALSE off.
 *
 * Result: none.
 */
void
xonxoff_proc(state)
int state;
{
    if(_inraw){
	if(state){
	    if(!(_raw_tty.c_iflag & IXON)){
		_raw_tty.c_iflag |= IXON;
		tcsetattr (STDIN_FD, TCSADRAIN, &_raw_tty);
	    }
	}
	else{
	    if(_raw_tty.c_iflag & IXON){
		_raw_tty.c_iflag &= ~IXON;	/* turn off ^S/^Q on input   */
		tcsetattr (STDIN_FD, TCSADRAIN, &_raw_tty);
	    }
	}
    }
}


/*
 *  Set up the tty driver to do LF->CR translation
 *
 * Args: state -- True to turn on translation, false to write raw LF's
 *
 * Result: none.
 */
void
crlf_proc(state)
int state;
{
    if(_inraw){
	if(state){				/* turn ON NL->CR on output */
	    if(!(_raw_tty.c_oflag & ONLCR)){
		_raw_tty.c_oflag |= ONLCR;
		tcsetattr (STDIN_FD, TCSADRAIN, &_raw_tty);
	    }
	}
	else{					/* turn OFF NL-CR on output */
	    if(_raw_tty.c_oflag & ONLCR){
		_raw_tty.c_oflag &= ~ONLCR;
		tcsetattr (STDIN_FD, TCSADRAIN, &_raw_tty);
	    }
	}
    }
}


/*
 *  Set up the tty driver to hanle interrupt char
 *
 * Args: state -- True to turn on interrupt char, false to not
 *
 * Result: tty driver that'll send us SIGINT or not
 */
void
intr_proc(state)
int state;
{
    if(_inraw){
	if(state){
	    _raw_tty.c_lflag |= ISIG;		/* enable signals */
	    tcsetattr(STDIN_FD, TCSADRAIN, &_raw_tty);
	}
	else{
	    _raw_tty.c_lflag &= ~ISIG;		/* disable signals */
	    tcsetattr(STDIN_FD, TCSADRAIN, &_raw_tty);
	}
    }
}
#endif


/*
 * Discard any pending input characters
 *
 * Args:  none
 *
 * Result: pending input buffer flushed
 */
void
flush_input()
{
#ifndef __riscos
    tcflush(STDIN_FD, TCIFLUSH);
#endif
}


/*
 * Turn off hi bit stripping
 */
void
bit_strip_off()
{
#ifndef __riscos
    _raw_tty.c_iflag &= ~ISTRIP;
    tcsetattr(STDIN_FD, TCSADRAIN, &_raw_tty);
#endif
}


/*
 * Turn off quit character (^\) if possible
 */
void
quit_char_off()
{
}


/*
 * Returns TRUE if tty is < 4800, 0 otherwise.
 */
int
ttisslow()
{
#ifdef __riscos
    return 0; /* TTY can be considered 'fast' on RISC OS */
#else
    return(cfgetospeed(&_raw_tty) < B4800);
#endif
}


/*
 * Program:	Display routines
 *
 *
 * Donn Cave
 * Networks and Distributed Computing
 * Computing and Communications
 * University of Washington
 * Administration Builiding, AG-44
 * Seattle, Washington, 98195, USA
 * Internet: donn@cac.washington.edu
 *
 * Please address all bugs and comments to "pine-bugs@cac.washington.edu"
 *
 *
 * Pine and Pico are registered trademarks of the University of Washington.
 * No commercial use of these trademarks may be made without prior written
 * permission of the University of Washington.
 *
 * Pine, Pico, and Pilot software and its included text are Copyright
 * 1989-2001 by the University of Washington.
 *
 * The full text of our legal notices is contained in the file called
 * CPYRIGHT, included with this distribution.
 *
 */
/*
 *      tinfo - substitute for tcap, on systems that have terminfo.
 */

#define	termdef	1			/* don't define "term" external */

#if defined(USE_TERMINFO)
extern char *tigetstr ();

#define	MARGIN	8
#define	SCRSIZ	64
#define	MROW	2

static int      tinfomove PROTO((int, int));
static int      tinfoeeol PROTO((void));
static int      tinfoeeop PROTO((void));
static int      tinfobeep PROTO((void));
static int      tinforev PROTO((int));
static int      tinfoopen PROTO((void));
static int      tinfoterminalinfo PROTO((int));
static int      tinfoclose PROTO((void));
static void     setup_dflt_esc_seq PROTO((void));
static void     tinfoinsert PROTO((int));
static void     tinfodelete PROTO((void));

extern int      tput();

/**
 ** Note: The tgoto calls should really be replaced by tparm calls for
 ** modern terminfo. tgoto(s, x, y) == tparm(s, y, x).
 **/

int	_tlines, _tcolumns;
char	*_clearscreen, *_moveto, *_up, *_down, *_right, *_left,
	*_setinverse, *_clearinverse,
	*_setunderline, *_clearunderline,
	*_setbold, *_clearallattr,	/* there is no clear only bold! */
	*_cleartoeoln, *_cleartoeos,
	*_deleteline,		/* delete line */
	*_insertline,		/* insert line */
	*_scrollregion,		/* define a scrolling region, vt100 */
	*_insertchar,		/* insert character, preferable to : */
	*_startinsert,		/* set insert mode and, */
	*_endinsert,		/* end insert mode */
	*_deletechar,		/* delete character */
	*_startdelete,		/* set delete mode and, */
	*_enddelete,		/* end delete mode */
	*_scrolldown,		/* scroll down */
	*_scrollup,		/* scroll up */
	*_termcap_init,		/* string to start termcap */
        *_termcap_end,		/* string to end termcap */
	*_op, *_oc, *_setaf, *_setab, *_setf, *_setb, *_scp;
int	_colors, _pairs, _bce;
char     term_name[40];

TERM term = {
        NROW-1,
        NCOL,
	MARGIN,
	SCRSIZ,
	MROW,
        tinfoopen,
        tinfoterminalinfo,
        tinfoclose,
        ttgetc,
        ttputc,
        ttflush,
        tinfomove,
        tinfoeeol,
        tinfoeeop,
        tinfobeep,
        tinforev
};


/*
 * Add default keypad sequences to the trie.
 */
static void
setup_dflt_esc_seq()
{
    /*
     * this is sort of a hack [no kidding], but it allows us to use
     * the function keys on pc's running telnet
     */

    /*
     * UW-NDC/UCS vt10[02] application mode.
     */
    kpinsert("\033OP", F1, 1);
    kpinsert("\033OQ", F2, 1);
    kpinsert("\033OR", F3, 1);
    kpinsert("\033OS", F4, 1);
    kpinsert("\033Op", F5, 1);
    kpinsert("\033Oq", F6, 1);
    kpinsert("\033Or", F7, 1);
    kpinsert("\033Os", F8, 1);
    kpinsert("\033Ot", F9, 1);
    kpinsert("\033Ou", F10, 1);
    kpinsert("\033Ov", F11, 1);
    kpinsert("\033Ow", F12, 1);

    /*
     * DEC vt100, ANSI and cursor key mode.
     */
    kpinsert("\033OA", KEY_UP, 1);
    kpinsert("\033OB", KEY_DOWN, 1);
    kpinsert("\033OC", KEY_RIGHT, 1);
    kpinsert("\033OD", KEY_LEFT, 1);

    /*
     * special keypad functions
     */
    kpinsert("\033[4J", KEY_PGUP, 1);
    kpinsert("\033[3J", KEY_PGDN, 1);
    kpinsert("\033[2J", KEY_HOME, 1);
    kpinsert("\033[N",  KEY_END, 1);

    /*
     * vt220?
     */
    kpinsert("\033[5~", KEY_PGUP, 1);
    kpinsert("\033[6~", KEY_PGDN, 1);
    kpinsert("\033[1~", KEY_HOME, 1);
    kpinsert("\033[4~", KEY_END, 1);

    /*
     * konsole, XTerm (XFree 4.x.x) keyboard setting
     */
    kpinsert("\033[H", KEY_HOME, 1);
    kpinsert("\033[F", KEY_END, 1);

    /*
     * gnome-terminal 2.6.0, don't know why it
     * changed from 2.2.1
     */
    kpinsert("\033OH", KEY_HOME, 1);
    kpinsert("\033OF", KEY_END, 1);

    /*
     * "\033[2~" was common for KEY_HOME in a quick survey
     *  of terminals (though typically the Insert key).
     *  Teraterm 2.33 sends the following escape sequences,
     *  which is quite incompatible with everything
     *  else:
     *    Home: "\033[2~" End: "\033[5~" PgUp: "\033[3~"
     *    PgDn: "\033[6~"
     *  The best thing to do would be to fix TeraTerm
     *  keymappings or to tweak terminfo.
     */

    /*
     * ANSI mode.
     */
    kpinsert("\033[=a", F1, 1);
    kpinsert("\033[=b", F2, 1);
    kpinsert("\033[=c", F3, 1);
    kpinsert("\033[=d", F4, 1);
    kpinsert("\033[=e", F5, 1);
    kpinsert("\033[=f", F6, 1);
    kpinsert("\033[=g", F7, 1);
    kpinsert("\033[=h", F8, 1);
    kpinsert("\033[=i", F9, 1);
    kpinsert("\033[=j", F10, 1);
    kpinsert("\033[=k", F11, 1);
    kpinsert("\033[=l", F12, 1);

    /*
     * DEC vt100, ANSI and cursor key mode reset.
     */
    kpinsert("\033[A", KEY_UP, 1);
    kpinsert("\033[B", KEY_DOWN, 1);
    kpinsert("\033[C", KEY_RIGHT, 1);
    kpinsert("\033[D", KEY_LEFT, 1);

    /*
     * DEC vt52 mode.
     */
    kpinsert("\033A", KEY_UP, 1);
    kpinsert("\033B", KEY_DOWN, 1);
    kpinsert("\033C", KEY_RIGHT, 1);
    kpinsert("\033D", KEY_LEFT, 1);

    /*
     * DEC vt52 application keys, and some Zenith 19.
     */
    kpinsert("\033?r", KEY_DOWN, 1);
    kpinsert("\033?t", KEY_LEFT, 1);
    kpinsert("\033?v", KEY_RIGHT, 1);
    kpinsert("\033?x", KEY_UP, 1);

    /*
     * Sun Console sequences.
     */
    kpinsert("\033[1",   KEY_SWALLOW_Z, 1);
    kpinsert("\033[215", KEY_SWAL_UP, 1);
    kpinsert("\033[217", KEY_SWAL_LEFT, 1);
    kpinsert("\033[219", KEY_SWAL_RIGHT, 1);
    kpinsert("\033[221", KEY_SWAL_DOWN, 1);

    /*
     * Kermit App Prog Cmd, gobble until ESC \ (kermit should intercept this)
     */
    kpinsert("\033_", KEY_KERMIT, 1);

    /*
     * Fake a control character.
     */
    kpinsert("\033\033", KEY_DOUBLE_ESC, 1);
}


static int
tinfoterminalinfo(termcap_wins)
    int termcap_wins;
{
#ifndef __riscos
    char   *_ku, *_kd, *_kl, *_kr,
	   *_kppu, *_kppd, *_kphome, *_kpend, *_kpdel,
	   *_kf1, *_kf2, *_kf3, *_kf4, *_kf5, *_kf6,
	   *_kf7, *_kf8, *_kf9, *_kf10, *_kf11, *_kf12;
    char   *ttnm;

    if (Pmaster) {
	/*
	 *		setupterm() automatically retrieves the value
	 *		of the TERM variable.
	 */
	int err;
	ttnm = getenv("TERM");
	if(!ttnm)
	  return(-1);

	strcpy(term_name, ttnm);
	setupterm (0, 1, &err);
	if (err != 1) return(err-2);
    }
    else {
	/*
	 *		setupterm() issues a message and exits, if the
	 *		terminfo data base is gone or the term type is
	 *		unknown, if arg2 is 0.
	 */
	setupterm (0, 1, 0);
    }

    _clearscreen	= tigetstr("clear");
    _moveto		= tigetstr("cup");
    _up			= tigetstr("cuu1");
    _down		= tigetstr("cud1");
    _right		= tigetstr("cuf1");
    _left		= tigetstr("cub1");
    _setinverse		= tigetstr("smso");
    _clearinverse	= tigetstr("rmso");
    _setunderline	= tigetstr("smul");
    _clearunderline	= tigetstr("rmul");
    _setbold		= tigetstr("bold");
    _clearallattr	= tigetstr("sgr0");
    _cleartoeoln	= tigetstr("el");
    _cleartoeos		= tigetstr("ed");
    _deletechar		= tigetstr("dch1");
    _insertchar		= tigetstr("ich1");
    _startinsert	= tigetstr("smir");
    _endinsert		= tigetstr("rmir");
    _deleteline		= tigetstr("dl1");
    _insertline		= tigetstr("il1");
    _scrollregion	= tigetstr("csr");
    _scrolldown		= tigetstr("ind");
    _scrollup		= tigetstr("ri");
    _termcap_init	= tigetstr("smcup");
    _termcap_end	= tigetstr("rmcup");
    _startdelete	= tigetstr("smdc");
    _enddelete		= tigetstr("rmdc");
    _ku			= tigetstr("kcuu1");
    _kd			= tigetstr("kcud1");
    _kl			= tigetstr("kcub1");
    _kr			= tigetstr("kcuf1");
    _kppu		= tigetstr("kpp");
    _kppd		= tigetstr("knp");
    _kphome		= tigetstr("khome");
    _kpend		= tigetstr("kend");
    _kpdel		= tigetstr("kdch1");
    _kf1		= tigetstr("kf1");
    _kf2		= tigetstr("kf2");
    _kf3		= tigetstr("kf3");
    _kf4		= tigetstr("kf4");
    _kf5		= tigetstr("kf5");
    _kf6		= tigetstr("kf6");
    _kf7		= tigetstr("kf7");
    _kf8		= tigetstr("kf8");
    _kf9		= tigetstr("kf9");
    _kf10		= tigetstr("kf10");
    _kf11		= tigetstr("kf11");
    _kf12		= tigetstr("kf12");

    _colors		= tigetnum("colors");
    _pairs		= tigetnum("pairs");
    _setaf		= tigetstr("setaf");
    _setab		= tigetstr("setab");
    _setf		= tigetstr("setf");
    _setb		= tigetstr("setb");
    _scp		= tigetstr("scp");
    _op			= tigetstr("op");
    _oc			= tigetstr("oc");
    _bce		= tigetflag("bce");

    _tlines = tigetnum("lines");
    if(_tlines == -1){
	char *er;
	int   rr;

	/* tigetnum failed, try $LINES */
	er = getenv("LINES");
	if(er && (rr = atoi(er)) > 0)
	  _tlines = rr;
    }

    _tcolumns = tigetnum("cols");
    if(_tcolumns == -1){
	char *ec;
	int   cc;

	/* tigetnum failed, try $COLUMNS */
	ec = getenv("COLUMNS");
	if(ec && (cc = atoi(ec)) > 0)
	  _tcolumns = cc;
    }

    /*
     * Add default keypad sequences to the trie.
     * Since these come first, they will override any conflicting termcap
     * or terminfo escape sequences defined below.  An escape sequence is
     * considered conflicting if one is a prefix of the other.
     * So, without TERMCAP_WINS, there will likely be some termcap/terminfo
     * escape sequences that don't work, because they conflict with default
     * sequences defined here.
     */
    if(!termcap_wins)
      setup_dflt_esc_seq();

    /*
     * add termcap/info escape sequences to the trie...
     */

    if(_ku != NULL && _kd != NULL && _kl != NULL && _kr != NULL){
	kpinsert(_ku, KEY_UP, termcap_wins);
	kpinsert(_kd, KEY_DOWN, termcap_wins);
	kpinsert(_kl, KEY_LEFT, termcap_wins);
	kpinsert(_kr, KEY_RIGHT, termcap_wins);
    }

    if(_kppu != NULL && _kppd != NULL){
	kpinsert(_kppu, KEY_PGUP, termcap_wins);
	kpinsert(_kppd, KEY_PGDN, termcap_wins);
    }

    kpinsert(_kphome, KEY_HOME, termcap_wins);
    kpinsert(_kpend,  KEY_END, termcap_wins);
    kpinsert(_kpdel,  KEY_DEL, termcap_wins);

    kpinsert(_kf1,  F1, termcap_wins);
    kpinsert(_kf2,  F2, termcap_wins);
    kpinsert(_kf3,  F3, termcap_wins);
    kpinsert(_kf4,  F4, termcap_wins);
    kpinsert(_kf5,  F5, termcap_wins);
    kpinsert(_kf6,  F6, termcap_wins);
    kpinsert(_kf7,  F7, termcap_wins);
    kpinsert(_kf8,  F8, termcap_wins);
    kpinsert(_kf9,  F9, termcap_wins);
    kpinsert(_kf10, F10, termcap_wins);
    kpinsert(_kf11, F11, termcap_wins);
    kpinsert(_kf12, F12, termcap_wins);

#else
    int     row, col;
    char   *er;
    char   *ec;
    int     cc,rr;

    int sregs[11];

    /*
     * determine the terminal's communication speed and decide
     * if we need to do optimization ...
     */
    optimize = 0;/*ttisslow();*/

    if (taskwindow())
    {
      row=-1; col=-1;
      revexist=FALSE;
    }
    else
    {
      int vars[]={ 256, 257, -1 };
      if (_swix(OS_ReadVduVariables,_INR(0,1),&vars[0],&vars[0]))
      {
        printf("Failed to read window size\n");
      }
      else
      {
          col=vars[0];
          row=vars[1];
      }

      revexist=TRUE;
    }
    /* Now check if there is an override on these */
    er = getenv("Pico$Lines");
    if (er==NULL && taskwindow()) er = getenv("Lines");
    if(er != NULL && (rr = atoi(er)) > 0)
      row = rr-1;
    ec = getenv("Pico$Columns");
    if (ec==NULL && taskwindow()) ec = getenv("Columns");
    if (ec != NULL && (cc = atoi(ec)) > 0)
      col = cc;

    ttgetwinsz(&row, &col);
    term.t_nrow = (short) row;
    term.t_ncol = (short) col;

#endif
    /*
     * Add default keypad sequences to the trie.
     * Since these come after the termcap/terminfo escape sequences above,
     * the termcap/info sequences will override any conflicting default
     * escape sequences defined here.
     * So, with TERMCAP_WINS, some of the default sequences will be missing.
     * This means that you'd better get all of your termcap/terminfo entries
     * correct if you define TERMCAP_WINS.
     */
#ifndef __riscos
    if(termcap_wins)
#endif
      setup_dflt_esc_seq();

    if(Pmaster)
      return(0);
    else
      return(TRUE);
}

static int
tinfoopen()
{
#ifndef __riscos
    int     row, col;

    /*
     * determine the terminal's communication speed and decide
     * if we need to do optimization ...
     */
    optimize = ttisslow();

    col = _tcolumns;
    row = _tlines;
    if(row >= 0)
      row--;

    ttgetwinsz(&row, &col);
    term.t_nrow = (short) row;
    term.t_ncol = (short) col;

    eolexist = (_cleartoeoln != NULL);	/* able to use clear to EOL? */
    revexist = (_setinverse != NULL);
    if(_deletechar == NULL && (_startdelete == NULL || _enddelete == NULL))
      delchar = FALSE;
    if(_insertchar == NULL && (_startinsert == NULL || _endinsert == NULL))
      inschar = FALSE;
    if((_scrollregion == NULL || _scrolldown == NULL || _scrollup == NULL)
       && (_deleteline == NULL || _insertline == NULL))
      scrollexist = FALSE;

    if(_clearscreen == NULL || _moveto == NULL || _up == NULL){
	if(Pmaster == NULL){
	    puts("Incomplete terminfo entry\n");
	    exit(1);
	}
    }
#else
    /* eolexist = !taskwindow();	/-* able to use clear to EOL? -*/
    eolexist = TRUE;
    revexist = TRUE;            /* Pretend we can do reverse */
    scrollexist = FALSE;        /* No scroll operations */
    inschar = FALSE;            /* No insert character */
    delchar = FALSE;            /* No insert character */
#endif

    ttopen();

#ifndef __riscos
    if(_termcap_init && !Pmaster) {
	putpad(_termcap_init);		/* any init terminfo requires */
	if (_scrollregion)
	  putpad(tgoto(_scrollregion, term.t_nrow, 0)) ;
    }
#endif

    /*
     * Initialize UW-modified NCSA telnet to use its functionkeys
     */
#ifndef __riscos
    if(gmode&MDFKEY && Pmaster == NULL && (gmode&MDUSEANSI))
#else
    if(gmode&MDFKEY && Pmaster == NULL)
#endif
      puts("\033[99h");

    /* return ignored */
    return(0);
}


static tinfoclose()
{
    if(!Pmaster){
#ifndef __riscos
	if((gmode&MDFKEY) && (gmode&MDUSEANSI))
#else
	if(gmode&MDFKEY)
#endif
	  puts("\033[99l");		/* reset UW-NCSA telnet keys */

#ifndef __riscos
	if(_termcap_end)		/* any clean up terminfo requires */
	  putpad(_termcap_end);
#endif
    }

    ttclose();

    /* return ignored */
    return(0);
}


/*
 * tinfoinsert - insert a character at the current character position.
 *               _insertchar takes precedence.
 */
static void
tinfoinsert(ch)
register int	ch;
{
#ifndef __riscos
    if(_insertchar != NULL){
	putpad(_insertchar);
	ttputc(ch);
    }
    else{
	putpad(_startinsert);
	ttputc(ch);
	putpad(_endinsert);
    }
#else
    _swix(OS_Write0,_IN(0),"tinfoinsert not supported");
#endif
}


/*
 * tinfodelete - delete a character at the current character position.
 */
static void
tinfodelete()
{
#ifndef __riscos
    if(_startdelete == NULL && _enddelete == NULL)
      putpad(_deletechar);
    else{
	putpad(_startdelete);
	putpad(_deletechar);
	putpad(_enddelete);
    }
#else
    _swix(OS_Write0,_IN(0),"tinfodelete not supported");
#endif
}


/*
 * o_scrolldown() - open a line at the given row position.
 *               use either region scrolling or deleteline/insertline
 *               to open a new line.
 */
o_scrolldown(row, n)
register int row;
register int n;
{
#ifdef __riscos
  _swix(OS_Write0,_IN(0),"o_scrolldown should not get called\n");
#else
    register int i;

    if(_scrollregion != NULL){
	putpad(tgoto(_scrollregion, term.t_nrow - (term.t_mrow+1), row));
	tinfomove(row, 0);
	for(i = 0; i < n; i++)
	  putpad((_scrollup != NULL && *_scrollup != '\0') ? _scrollup : "\n" );
	putpad(tgoto(_scrollregion, term.t_nrow, 0));
	tinfomove(row, 0);
    }
    else{
	/*
	 * this code causes a jiggly motion of the keymenu when scrolling
	 */
	for(i = 0; i < n; i++){
	    tinfomove(term.t_nrow - (term.t_mrow+1), 0);
	    putpad(_deleteline);
	    tinfomove(row, 0);
	    putpad(_insertline);
	}
#ifdef	NOWIGGLYLINES
	/*
	 * this code causes a sweeping motion up and down the display
	 */
	tinfomove(term.t_nrow - term.t_mrow - n, 0);
	for(i = 0; i < n; i++)
	  putpad(_deleteline);
	tinfomove(row, 0);
	for(i = 0; i < n; i++)
	  putpad(_insertline);
#endif
    }
#endif /* __riscos */

    /* return ignored */
    return(0);
}


/*
 * o_scrollup() - open a line at the given row position.
 *               use either region scrolling or deleteline/insertline
 *               to open a new line.
 */
o_scrollup(row, n)
register int row;
register int n;
{
#ifdef __riscos
  _swix(OS_Write0,_IN(0),"o_scrollup should not get called\n");
#else
    register int i;

    if(_scrollregion != NULL){
	putpad(tgoto(_scrollregion, term.t_nrow - (term.t_mrow+1), row));
	/* setting scrolling region moves cursor to home */
	tinfomove(term.t_nrow-(term.t_mrow+1), 0);
	for(i = 0;i < n; i++)
	  putpad((_scrolldown == NULL || _scrolldown[0] == '\0') ? "\n"
								 : _scrolldown);
	putpad(tgoto(_scrollregion, term.t_nrow, 0));
	tinfomove(2, 0);
    }
    else{
	for(i = 0; i < n; i++){
	    tinfomove(row, 0);
	    putpad(_deleteline);
	    tinfomove(term.t_nrow - (term.t_mrow+1), 0);
	    putpad(_insertline);
	}
#ifdef  NOWIGGLYLINES
	/* see note above */
	tinfomove(row, 0);
	for(i = 0; i < n; i++)
	  putpad(_deleteline);
	tinfomove(term.t_nrow - term.t_mrow - n, 0);
	for(i = 0;i < n; i++)
	  putpad(_insertline);
#endif
    }
#endif

    /* return ignored */
    return(0);
}



/*
 * o_insert - use terminfo to optimized character insert
 *            returns: true if it optimized output, false otherwise
 */
int
o_insert(c)
int c;
{
#ifndef __riscos
    if(inschar){
	tinfoinsert(c);
	return(1);			/* no problems! */
    }
#endif

    return(0);				/* can't do it. */
}


/*
 * o_delete - use terminfo to optimized character insert
 *            returns true if it optimized output, false otherwise
 */
o_delete()
{
#ifndef __riscos
    if(delchar){
	tinfodelete();
	return(1);			/* deleted, no problem! */
    }
#endif

    return(0);				/* no dice. */
}


#ifdef __riscos
void
ansiparm(n)
register int    n;
{
        register int    q;

        q = n/10;
        if (q != 0)
                ansiparm(q);
        _swix(OS_WriteC,_IN(0),((n%10) + '0'));
}
#endif


static tinfomove(row, col)
register int row, col;
{
#ifdef __riscos
  if (gmode & MDUSEANSI)
  {
    _swix(OS_WriteC,_IN(0),27);
    _swix(OS_WriteC,_IN(0),'[');
    ansiparm(row+1);
    _swix(OS_WriteC,_IN(0),';');
    ansiparm(col+1);
    _swix(OS_WriteC,_IN(0),'H');
  }
  else
  {
    _swix(OS_WriteC,_IN(0),31);
    _swix(OS_WriteC,_IN(0),col);
    _swix(OS_WriteC,_IN(0),row);
  }
#else
    putpad(tgoto(_moveto, col, row));
#endif

    /* return ignored */
    return(0);
}


static int tinfoeeol(void)
{
    int   c, starting_col, starting_line;
#ifndef __riscos
    char *last_bg_color;

    /*
     * If the terminal doesn't have back color erase, then we have to
     * erase manually to preserve the background color.
     */
    if(pico_usingcolor() && (!_bce || !_cleartoeoln)){
	extern int ttcol, ttrow;

	starting_col  = ttcol;
	starting_line = ttrow;
	last_bg_color = pico_get_last_bg_color();
	pico_set_nbg_color();
	for(c = ttcol; c < term.t_ncol; c++)
	  ttputc(' ');

        tinfomove(starting_line, starting_col);
	if(last_bg_color){
	    pico_set_bg_color(last_bg_color);
	    free(last_bg_color);
	}
    }
    else if(_cleartoeoln)
      putpad(_cleartoeoln);
#else
  if (gmode & MDUSEANSI)
  {
    _swix(OS_WriteC,_IN(0),27);
    _swix(OS_WriteC,_IN(0),'[');
    _swix(OS_WriteC,_IN(0),'K');
  }
  else
  {
    if (taskwindow())
    {
      extern int ttcol;
      int fromx;
      /* TaskWindows (at least Zap ones) don't support the 'delete to end of
         line' */
      int i;
      fromx  = ttcol;
      for (i=term.t_ncol; i>fromx; i--)
      {
        _swix(OS_WriteC,_IN(0),' ');
      }
      for (i=term.t_ncol; i>fromx; i--)
      {
        _swix(OS_WriteC,_IN(0),8);
      }
    }
    else
    {
      _swix(OS_WriteC,_IN(0),23);
      _swix(OS_WriteC,_IN(0),8);
      _swix(OS_WriteC,_IN(0),5);
      _swix(OS_WriteC,_IN(0),6);
      _swix(OS_WriteC,_IN(0),0);
      _swix(OS_WriteC,_IN(0),0);
      _swix(OS_WriteC,_IN(0),0);
      _swix(OS_WriteC,_IN(0),0);
      _swix(OS_WriteC,_IN(0),0);
      _swix(OS_WriteC,_IN(0),0);
    }
  }
#endif

    /* return ignored */
    return(0);
}


static int tinfoeeop(void)
{
#ifndef __riscos
    int i, starting_col, starting_row;

    /*
     * If the terminal doesn't have back color erase, then we have to
     * erase manually to preserve the background color.
     */
    if(pico_usingcolor() && (!_bce || !_cleartoeos)){
	extern int ttcol, ttrow;

	starting_col = ttcol;
	starting_row = ttrow;
	tinfoeeol();				  /* rest of this line */
	for(i = ttrow+1; i <= term.t_nrow; i++){  /* the remaining lines */
	    tinfomove(i, 0);
	    tinfoeeol();
	}

	tinfomove(starting_row, starting_col);
    }
    else if(_cleartoeos)
      putpad(_cleartoeos);
#else
  if (gmode & MDUSEANSI)
  {
    _swix(OS_WriteC,_IN(0),27);
    _swix(OS_WriteC,_IN(0),'[');
    _swix(OS_WriteC,_IN(0),'J');
  }
  else
  {
    _swix(OS_WriteC,_IN(0),12);
  }
#endif

    /* return ignored */
    return(0);
}


static tinforev(state)		/* change reverse video status */
int state;			/* FALSE = normal video, TRUE = rev video */
{
#ifdef __riscos
  static int cstate = FALSE;

  if(state == cstate)		/* no op if already set! */
    return(0);

  cstate = state;
  if (gmode & MDUSEANSI)
  {
    _swix(OS_WriteC,_IN(0),27);
    _swix(OS_WriteC,_IN(0),'[');
    _swix(OS_WriteC,_IN(0),(state ? '7': '0'));
    _swix(OS_WriteC,_IN(0),'m');
  }
  else
  {
    _swix(OS_WriteC,_IN(0),23);
    _swix(OS_WriteC,_IN(0),17);
    _swix(OS_WriteC,_IN(0),5);
    _swix(OS_WriteC,_IN(0),0);
    _swix(OS_WriteC,_IN(0),0);
    _swix(OS_WriteC,_IN(0),0);
    _swix(OS_WriteC,_IN(0),0);
    _swix(OS_WriteC,_IN(0),0);
    _swix(OS_WriteC,_IN(0),0);
    _swix(OS_WriteC,_IN(0),0);
  }
#else
    if(state)
      StartInverse();
    else
      EndInverse();
#endif
    return(1);
}


/*********************************************** <c> Gerph *********
 Function:     taskwindow()
 Description:  are we in a taskwindow ?
 Parameters:   none
 Returns:      TRUE if we are, FALSE otherwise
 ******************************************************************/
int taskwindow(void)
{
  int handle;
  _swix(TaskWindow_TaskInfo,_IN(0)|_OUT(0),0,&handle);
  return (handle!=0) ? TRUE : FALSE;
}


static tinfobeep()
{
#ifdef __riscos
    _swix(OS_WriteC,_IN(0),7);
#else
    ttputc(BELL);
#endif

    /* return ignored */
    return(0);
}

#ifndef __riscos
static void
putpad(str)
char    *str;
{
    tputs(str, 1, ttputc);
}
#endif

#endif /* USE_TERMINFO */


