#if	!defined(lint) && !defined(DOS)
static char rcsid[] = "$Id: tcap.c,v 4.26 1996/03/15 07:41:11 hubert Exp $";
#endif
/*
 * Program:	Display routines
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
 */
/*	tcap:	Unix V5, V7 and BS4.2 Termcap video driver
		for MicroEMACS
*/

#define	termdef	1			/* don't define "term" external */

#include	<stdio.h>
#include        <signal.h>
#include	"osdep.h"
#include        "pico.h"
#include	"estruct.h"
#include        "edef.h"

#if TERMCAP
#define	MARGIN	8
#define	SCRSIZ	64
#define	MROW	2
#define BEL     0x07
#define ESC     0x1B

extern int      ttopen();
extern int      ttgetc();
extern int      ttputc();
extern int      ttflush();
extern int      ttclose();

static int      tcapmove();
static int      tcapeeol();
static int      tcapeeop();
static int      tcapbeep();
static int	tcaprev();
static int      tcapopen();
static int      tcapclose();
static void     setup_dflt_pico_esc_seq();

extern int      tput();
extern char     *tgoto();

#define TCAPSLEN 315
char tcapbuf[TCAPSLEN];
char *UP, PC, *CM, *CE, *CL, *SO, *SE;
/*
 * PICO extentions
 */
char	*DL,			/* delete line */
	*AL,			/* insert line */
	*CS,			/* define a scrolling region, vt100 */
	*IC,			/* insert character, preferable to : */
	*IM,			/* set insert mode and, */
	*EI,			/* end insert mode */
	*DC,			/* delete character */
	*DM,			/* set delete mode and, */
	*ED,			/* end delete mode */
	*SF,			/* scroll text up */
	*SR,			/* scroll text down */
	*TI,			/* string to start termcap */
        *TE;			/* string to end termcap */


TERM term = {
        NROW-1,
        NCOL,
	MARGIN,
	SCRSIZ,
	MROW,
        tcapopen,
        tcapclose,
        ttgetc,
        ttputc,
        ttflush,
        tcapmove,
        tcapeeol,
        tcapeeop,
        tcapbeep,
        tcaprev
};


/*
 * Add default keypad sequences to the trie.
 */
static void
setup_dflt_pico_esc_seq()
{
    /*
     * this is sort of a hack, but it allows us to use
     * the function keys on pc's running telnet
     */

    /*
     * UW-NDC/UCS vt10[02] application mode.
     */
    kpinsert(&pico_kbesc, "\033OP", F1);
    kpinsert(&pico_kbesc, "\033OQ", F2);
    kpinsert(&pico_kbesc, "\033OR", F3);
    kpinsert(&pico_kbesc, "\033OS", F4);
    kpinsert(&pico_kbesc, "\033Op", F5);
    kpinsert(&pico_kbesc, "\033Oq", F6);
    kpinsert(&pico_kbesc, "\033Or", F7);
    kpinsert(&pico_kbesc, "\033Os", F8);
    kpinsert(&pico_kbesc, "\033Ot", F9);
    kpinsert(&pico_kbesc, "\033Ou", F10);
    kpinsert(&pico_kbesc, "\033Ov", F11);
    kpinsert(&pico_kbesc, "\033Ow", F12);

    /*
     * DC vt100, ANSI and cursor key mode.
     */
    kpinsert(&pico_kbesc, "\033OA", K_PAD_UP);
    kpinsert(&pico_kbesc, "\033OB", K_PAD_DOWN);
    kpinsert(&pico_kbesc, "\033OC", K_PAD_RIGHT);
    kpinsert(&pico_kbesc, "\033OD", K_PAD_LEFT);

    /*
     * special keypad functions
     */
    kpinsert(&pico_kbesc, "\033[4J", K_PAD_PREVPAGE);
    kpinsert(&pico_kbesc, "\033[3J", K_PAD_NEXTPAGE);
    kpinsert(&pico_kbesc, "\033[2J", K_PAD_HOME);
    kpinsert(&pico_kbesc, "\033[N",  K_PAD_END);

    /*
     * ANSI mode.
     */
    kpinsert(&pico_kbesc, "\033[=a", F1);
    kpinsert(&pico_kbesc, "\033[=b", F2);
    kpinsert(&pico_kbesc, "\033[=c", F3);
    kpinsert(&pico_kbesc, "\033[=d", F4);
    kpinsert(&pico_kbesc, "\033[=e", F5);
    kpinsert(&pico_kbesc, "\033[=f", F6);
    kpinsert(&pico_kbesc, "\033[=g", F7);
    kpinsert(&pico_kbesc, "\033[=h", F8);
    kpinsert(&pico_kbesc, "\033[=i", F9);
    kpinsert(&pico_kbesc, "\033[=j", F10);
    kpinsert(&pico_kbesc, "\033[=k", F11);
    kpinsert(&pico_kbesc, "\033[=l", F12);

    /*
     * DEC vt100, ANSI, cursor key mode reset.
     */
    kpinsert(&pico_kbesc, "\033[A", K_PAD_UP);
    kpinsert(&pico_kbesc, "\033[B", K_PAD_DOWN);
    kpinsert(&pico_kbesc, "\033[C", K_PAD_RIGHT);
    kpinsert(&pico_kbesc, "\033[D", K_PAD_LEFT);

    /*
     * DEC vt52 mode.
     */
    kpinsert(&pico_kbesc, "\033A", K_PAD_UP);
    kpinsert(&pico_kbesc, "\033B", K_PAD_DOWN);
    kpinsert(&pico_kbesc, "\033C", K_PAD_RIGHT);
    kpinsert(&pico_kbesc, "\033D", K_PAD_LEFT);

    /*
     * DEC vt52 application keys, and some Zenith 19.
     */
    kpinsert(&pico_kbesc, "\033?r", K_PAD_DOWN);
    kpinsert(&pico_kbesc, "\033?t", K_PAD_LEFT);
    kpinsert(&pico_kbesc, "\033?v", K_PAD_RIGHT);
    kpinsert(&pico_kbesc, "\033?x", K_PAD_UP);

    /*
     * Sun Console sequences.
     */
    kpinsert(&pico_kbesc, "\033[1",   K_SWALLOW_TIL_Z);
    kpinsert(&pico_kbesc, "\033[215", K_SWALLOW_UP);
    kpinsert(&pico_kbesc, "\033[217", K_SWALLOW_LEFT);
    kpinsert(&pico_kbesc, "\033[219", K_SWALLOW_RIGHT);
    kpinsert(&pico_kbesc, "\033[221", K_SWALLOW_DOWN);

    /*
     * Kermit App Prog Cmd, gobble until ESC \ (kermit should intercept this)
     */
    kpinsert(&pico_kbesc, "\033_", K_KERMIT);

    /*
     * Fake a control character.
     */
    kpinsert(&pico_kbesc, "\033\033", K_DOUBLE_ESC);
}


static int
tcapopen()
{
    char   *t, *p, *tgetstr();
    char    tcbuf[1024];
    char   *tv_stype;
    char    err_str[72];
    char   *getenv();
    int     row, col;
    char   *KU, *KD, *KL, *KR,
	   *KPPU, *KPPD, *KPHOME, *KPEND, *KPDEL,
	   *KF1, *KF2, *KF3, *KF4, *KF5, *KF6,
	   *KF7, *KF8, *KF9, *KF10, *KF11, *KF12;

    /*
     * determine the terminal's communication speed and decide
     * if we need to do optimization ...
     */
    optimize = ttisslow();

    if ((tv_stype = getenv("TERM")) == NULL){
	if(Pmaster){
	    return(FALSE);
	}
	else{
	    puts("Environment variable TERM not defined!");
	    exit(1);
	}
    }

    if((tgetent(tcbuf, tv_stype)) != 1){
	if(Pmaster){
	    return(FALSE);
	}
	else{
	    sprintf(err_str, "Unknown terminal type %s!", tv_stype);
	    puts(err_str);
	    exit(1);
	}
    }

    p = tcapbuf;
    t = tgetstr("pc", &p);
    if(t)
      PC = *t;

    CL = tgetstr("cl", &p);
    CM = tgetstr("cm", &p);
    CE = tgetstr("ce", &p);
    UP = tgetstr("up", &p);
    SE = tgetstr("se", &p);
    SO = tgetstr("so", &p);
    DL = tgetstr("dl", &p);
    AL = tgetstr("al", &p);
    CS = tgetstr("cs", &p);
    IC = tgetstr("ic", &p);
    IM = tgetstr("im", &p);
    EI = tgetstr("ei", &p);
    DC = tgetstr("dc", &p);
    DM = tgetstr("dm", &p);
    ED = tgetstr("ed", &p);
    SF = tgetstr("sf", &p);
    SR = tgetstr("sr", &p);
    TI = tgetstr("ti", &p);
    TE = tgetstr("te", &p);

    if (taskwindow())
    {
      SO=NULL; /* No reverse... */
      SE=NULL; /* ... and no reverse end */
    }

    row = tgetnum("li");
    if(row == -1){
	char *er;
	int   rr;

	/* tgetnum failed, try $LINES */
	er = getenv("LINES");
	if(er && (rr = atoi(er)) > 0)
	  row = rr;
    }
    if(row >= 0)
      row--;

    col = tgetnum("co");
    if(col == -1){
	char *ec;
	int   cc;

	/* tgetnum failed, try $COLUMNS */
	ec = getenv("COLUMNS");
	if(ec && (cc = atoi(ec)) > 0)
	  col = cc;
    }

    ttgetwinsz(&row, &col);
    term.t_nrow = (short) row;
    term.t_ncol = (short) col;

    eolexist = (CE != NULL);	/* will we be able to use clear to EOL? */
    revexist = (SO != NULL);
    if(DC == NULL && (DM == NULL || ED == NULL))
      delchar = FALSE;
    if(IC == NULL && (IM == NULL || EI == NULL))
      inschar = FALSE;
    if((CS==NULL || SF==NULL || SR==NULL) && (DL==NULL || AL==NULL))
      scrollexist = FALSE;

    if(CL == NULL || CM == NULL || UP == NULL){
	if(Pmaster == NULL){
	    puts("Incomplete termcap entry\n");
	    exit(1);
	}
    }
    else{
	KPPU   = tgetstr("kP", &p);
	KPPD   = tgetstr("kN", &p);
	KPHOME = tgetstr("kh", &p);
	KPEND  = tgetstr("kE", &p);
	KPDEL  = tgetstr("kD", &p);
	KU     = tgetstr("ku", &p);
	KD     = tgetstr("kd", &p);
	KL     = tgetstr("kl", &p);
	KR     = tgetstr("kr", &p);
	KF1    = tgetstr("k1", &p);
	KF2    = tgetstr("k2", &p);
	KF3    = tgetstr("k3", &p);
	KF4    = tgetstr("k4", &p);
	KF5    = tgetstr("k5", &p);
	KF6    = tgetstr("k6", &p);
	KF7    = tgetstr("k7", &p);
	KF8    = tgetstr("k8", &p);
	KF9    = tgetstr("k9", &p);
	if((KF10 = tgetstr("k;", &p)) == NULL)
	  KF10 = tgetstr("k0", &p);
	KF11   = tgetstr("F1", &p);
	KF12   = tgetstr("F2", &p);

#ifndef TERMCAP_WINS
	/*
	 * Add default keypad sequences to the trie.
	 * Since these come first, they will override any conflicting termcap
	 * or terminfo escape sequences defined below.  An escape sequence is
	 * considered conflicting if one is a prefix of the other.
	 * So, without TERMCAP_WINS, there will likely be some termcap/terminfo
	 * escape sequences that don't work, because they conflict with default
	 * sequences.
	 */
	setup_dflt_pico_esc_seq();
#endif /* !TERMCAP_WINS */

	/*
	 * add termcap escape sequences to the trie...
	 */
	if(KU != NULL && (KL != NULL && (KR != NULL && KD != NULL))){
	    kpinsert(&pico_kbesc, KU, K_PAD_UP);
	    kpinsert(&pico_kbesc, KD, K_PAD_DOWN);
	    kpinsert(&pico_kbesc, KL, K_PAD_LEFT);
	    kpinsert(&pico_kbesc, KR, K_PAD_RIGHT);
	}

	if(KPPU != NULL && KPPD != NULL){
	    kpinsert(&pico_kbesc, KPPU, K_PAD_PREVPAGE);
	    kpinsert(&pico_kbesc, KPPD, K_PAD_NEXTPAGE);
	}

	kpinsert(&pico_kbesc, KPHOME, K_PAD_HOME);
	kpinsert(&pico_kbesc, KPEND,  K_PAD_END);
	kpinsert(&pico_kbesc, KPDEL,  K_PAD_DELETE);

	kpinsert(&pico_kbesc, KF1,  F1);
	kpinsert(&pico_kbesc, KF2,  F2);
	kpinsert(&pico_kbesc, KF3,  F3);
	kpinsert(&pico_kbesc, KF4,  F4);
	kpinsert(&pico_kbesc, KF5,  F5);
	kpinsert(&pico_kbesc, KF6,  F6);
	kpinsert(&pico_kbesc, KF7,  F7);
	kpinsert(&pico_kbesc, KF8,  F8);
	kpinsert(&pico_kbesc, KF9,  F9);
	kpinsert(&pico_kbesc, KF10, F10);
	kpinsert(&pico_kbesc, KF11, F11);
	kpinsert(&pico_kbesc, KF12, F12);
    }

    /*
     * Initialize UW-modified NCSA telnet to use it's functionkeys
     */
    if(gmode&MDFKEY && Pmaster == NULL)
      puts("\033[99h");

#ifdef TERMCAP_WINS
    /*
     * Add default keypad sequences to the trie.
     * Since these come after the termcap/terminfo escape sequences above,
     * the termcap/info sequences will override any conflicting default
     * escape sequences defined here.
     * So, with TERMCAP_WINS, some of the default sequences will be missing.
     * This means that you'd better get all of your termcap/terminfo entries
     * correct if you define TERMCAP_WINS.
     */
    setup_dflt_pico_esc_seq();
#endif /* TERMCAP_WINS */

    if (p >= &tcapbuf[TCAPSLEN]){
	if(Pmaster == NULL){
	    puts("Terminal description too big!\n");
	    exit(1);
	}
    }

    ttopen();

    if(TI && !Pmaster) {
	putpad(TI);			/* any init termcap requires */
	if (CS)
	  putpad(tgoto(CS, term.t_nrow, 0)) ;
    }
}


static int
tcapclose()
{
    if(!Pmaster){
	if(gmode&MDFKEY)
	  puts("\033[99l");		/* reset UW-NCSA telnet keys */

	if(TE)				/* any cleanup termcap requires */
	  putpad(TE);
    }

    kbdestroy(pico_kbesc);		/* clean up key board sequence trie */
    pico_kbesc = NULL;
    ttclose();
}



/*
 * tcapinsert - insert a character at the current character position.
 *              IC takes precedence.
 */
tcapinsert(ch)
register char	ch;
{
    if(IC != NULL){
	putpad(IC);
	ttputc(ch);
    }
    else{
	putpad(IM);
	ttputc(ch);
	putpad(EI);
    }
}


/*
 * tcapdelete - delete a character at the current character position.
 */
tcapdelete()
{
    if(DM == NULL && ED == NULL)
      putpad(DC);
    else{
	putpad(DM);
	putpad(DC);
	putpad(ED);
    }
}


/*
 * o_scrolldown - open a line at the given row position.
 *                use either region scrolling or deleteline/insertline
 *                to open a new line.
 */
o_scrolldown(row, n)
register int row;
register int n;
{
    register int i;

    if(CS != NULL){
	putpad(tgoto(CS, term.t_nrow - (term.t_mrow+1), row));
	tcapmove(row, 0);
	for(i = 0; i < n; i++)
	  putpad( (SR != NULL && *SR != '\0') ? SR : "\n" );
	putpad(tgoto(CS, term.t_nrow, 0));
	tcapmove(row, 0);
    }
    else{
	/*
	 * this code causes a jiggly motion of the keymenu when scrolling
	 */
	for(i = 0; i < n; i++){
	    tcapmove(term.t_nrow - (term.t_mrow+1), 0);
	    putpad(DL);
	    tcapmove(row, 0);
	    putpad(AL);
	}
#ifdef	NOWIGGLYLINES
	/*
	 * this code causes a sweeping motion up and down the display
	 */
	tcapmove(term.t_nrow - term.t_mrow - n, 0);
	for(i = 0; i < n; i++)
	  putpad(DL);
	tcapmove(row, 0);
	for(i = 0; i < n; i++)
	  putpad(AL);
#endif
    }
}


/*
 * o_scrollup - open a line at the given row position.
 *              use either region scrolling or deleteline/insertline
 *              to open a new line.
 */
o_scrollup(row, n)
register int row;
register int n;
{
    register int i;

    if(CS != NULL){
	putpad(tgoto(CS, term.t_nrow - (term.t_mrow+1), row));
	/* setting scrolling region moves cursor to home */
	tcapmove(term.t_nrow-(term.t_mrow+1), 0);
	for(i = 0;i < n; i++)
	  putpad((SF == NULL || SF[0] == '\0') ? "\n" : SF);
	putpad(tgoto(CS, term.t_nrow, 0));
	tcapmove(2, 0);
    }
    else{
	for(i = 0; i < n; i++){
	    tcapmove(row, 0);
	    putpad(DL);
	    tcapmove(term.t_nrow - (term.t_mrow+1), 0);
	    putpad(AL);
	}
#ifdef  NOWIGGLYLINES
	/* see note above */
	tcapmove(row, 0);
	for(i = 0; i < n; i++)
	  putpad(DL);
	tcapmove(term.t_nrow - term.t_mrow - n, 0);
	for(i = 0;i < n; i++)
	  putpad(AL);
#endif
    }
}


/*
 * o_insert - use termcap info to optimized character insert
 *            returns: true if it optimized output, false otherwise
 */
o_insert(c)
char c;
{
    if(inschar){
	tcapinsert(c);
	return(1);			/* no problems! */
    }

    return(0);				/* can't do it. */
}


/*
 * o_delete - use termcap info to optimized character insert
 *            returns true if it optimized output, false otherwise
 */
o_delete()
{
    if(delchar){
	tcapdelete();
	return(1);			/* deleted, no problem! */
    }

    return(0);				/* no dice. */
}


static int
tcapmove(row, col)
register int row, col;
{
    putpad(tgoto(CM, col, row));
}


static int
tcapeeol()
{
    putpad(CE);
}


static int
tcapeeop()
{
    putpad(CL);
}


static int
tcaprev(state)		/* change reverse video status */
int state;	        /* FALSE = normal video, TRUE = reverse video */
{
    static int cstate = FALSE;

    if(state == cstate)		/* no op if already set! */
      return(0);

    if(cstate = state){		/* remember last setting */
	if (SO != NULL)
	  putpad(SO);
    }
    else{
	if (SE != NULL)
	  putpad(SE);
    }
}


static int
tcapbeep()
{
    ttputc(BEL);
}


putpad(str)
char    *str;
{
    tputs(str, 1, ttputc);
}


putnpad(str, n)
char    *str;
{
    tputs(str, n, ttputc);
}

#else

hello()
{
}

#endif /* TERMCAP */
