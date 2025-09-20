# bhuudo.github.io
<html>
<pre>
/***************************************************************************
 *  Original Diku Mud copyright (C) 1990, 1991 by Sebastian Hammer,        *
 *  Michael Seifert, Hans Henrik St{rfeldt, Tom Madsen, and Katja Nyboe.   *
 *                                                                         *
 *  Merc Diku Mud improvments copyright (C) 1992, 1993 by Michael          *
 *  Chastain, Michael Quan, and Mitchell Tse.                              *
 *                                                                         *
 *  In order to use any part of this Merc Diku Mud, you must comply with   *
 *  both the original Diku license in 'license.doc' as well the Merc       *
 *  license in 'license.txt'.  In particular, you may not remove either of *
 *  these copyright notices.                                               *
 *                                                                         *
 *  Much time and thought has gone into this software and you are          *
 *  benefitting.  We hope that you share your changes too.  What goes      *
 *  around, comes around.                                                  *
 ***************************************************************************/

#if defined(macintosh)
#include <types.h>
#else
#include <sys/types.h>
//#include <linux/types.h>
#endif
#include <ctype.h>
//#include <linux/ctype.h>
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <time.h>
#include "merc.h"



/*
 * Local functions.
 */
bool	is_note_to	args( ( CHAR_DATA *ch, NOTE_DATA *pnote ) );
void	note_attach	args( ( CHAR_DATA *ch ) );
void	note_remove	args( ( CHAR_DATA *ch, NOTE_DATA *pnote ) );
void	talk_channel	args( ( CHAR_DATA *ch, char *argument,
			    int channel, const char *verb ) );



bool is_note_to( CHAR_DATA *ch, NOTE_DATA *pnote )
{
    if ( !str_cmp( ch->name, pnote->sender ) )
	return TRUE;

    if ( is_name( "all", pnote->to_list ) )
	return TRUE;

    if ( IS_HERO(ch) && is_name( "immortal", pnote->to_list ) )
	return TRUE;

    if ( is_name( ch->name, pnote->to_list ) )
	return TRUE;

    return FALSE;
}



void note_attach( CHAR_DATA *ch )
{
    NOTE_DATA *pnote;

    if ( ch->pnote != NULL )
	return;

    if ( note_free == NULL )
    {
	pnote	  = alloc_perm( sizeof(*ch->pnote) );
    }
    else
    {
	pnote	  = note_free;
	note_free = note_free->next;
    }

    pnote->next		= NULL;
    pnote->sender	= str_dup( ch->name );
    pnote->date		= str_dup( "" );
    pnote->to_list	= str_dup( "" );
    pnote->subject	= str_dup( "" );
    pnote->text		= str_dup( "" );
    ch->pnote		= pnote;
    return;
}



void note_remove( CHAR_DATA *ch, NOTE_DATA *pnote )
{
    char to_new[MAX_INPUT_LENGTH];
    char to_one[MAX_INPUT_LENGTH];
    FILE *fp;
    NOTE_DATA *prev;
    char *to_list;

    /*
     * Build a new to_list.
     * Strip out this recipient.
     */
    to_new[0]	= '\0';
    to_list	= pnote->to_list;
    while ( *to_list != '\0' )
    {
	to_list	= one_argument( to_list, to_one );
	if ( to_one[0] != '\0' && str_cmp( ch->name, to_one ) )
	{
	    strcat( to_new, " " );
	    strcat( to_new, to_one );
	}
    }

    /*
     * Just a simple recipient removal?
     */
    if ( str_cmp( ch->name, pnote->sender ) && to_new[0] != '\0' )
    {
	free_string( pnote->to_list );
	pnote->to_list = str_dup( to_new + 1 );
	return;
    }

    /*
     * Remove note from linked list.
     */
    if ( pnote == note_list )
    {
	note_list = pnote->next;
    }
    else
    {
	for ( prev = note_list; prev != NULL; prev = prev->next )
	{
	    if ( prev->next == pnote )
		break;
	}

	if ( prev == NULL )
	{
	    bug( "Note_remove: pnote not found.", 0 );
	    return;
	}

	prev->next = pnote->next;
    }

    free_string( pnote->text    );
    free_string( pnote->subject );
    free_string( pnote->to_list );
    free_string( pnote->date    );
    free_string( pnote->sender  );
    pnote->next	= note_free;
    note_free	= pnote;

    /*
     * Rewrite entire list.
     */
    fclose( fpReserve );
    if ( ( fp = fopen( NOTE_FILE, "w" ) ) == NULL )
    {
	perror( NOTE_FILE );
    }
    else
    {
	for ( pnote = note_list; pnote != NULL; pnote = pnote->next )
	{
	    fprintf( fp, "Sender  %s~\nDate    %s~\nTo      %s~\nSubject %s~\nText\n%s~\n\n",
		pnote->sender,
		pnote->date,
		pnote->to_list,
		pnote->subject,
		pnote->text
		);
	}
	fclose( fp );
    }
    fpReserve = fopen( NULL_FILE, "r" );
    return;
}



void do_note( CHAR_DATA *ch, char *argument )
{
    char buf[MAX_STRING_LENGTH];
    char arg[MAX_INPUT_LENGTH];
    NOTE_DATA *pnote;
    int vnum;
    int anum;

    if ( IS_NPC(ch) )
	return;

    argument = one_argument( argument, arg );
    smash_tilde( argument );

    if ( !str_cmp( arg, "list" ) )
    {
	vnum = 0;
	for ( pnote = note_list; pnote != NULL; pnote = pnote->next )
	{
	    if ( is_note_to( ch, pnote ) )
	    {
		sprintf( buf, "[%3d] %s: %s\r\n",
		    vnum, pnote->sender, pnote->subject );
		send_to_char( buf, ch );
		vnum++;
	    }
	}
	return;
    }

    if ( !str_cmp( arg, "read" ) )
    {
	bool fAll;

	if ( !str_cmp( argument, "all" ) )
	{
	    fAll = TRUE;
	    anum = 0;
	}
	else if ( is_number( argument ) )
	{
	    fAll = FALSE;
	    anum = atoi( argument );
	}
	else
	{
	    send_to_char( "Note read which number?\r\n", ch );
	    return;
	}

	vnum = 0;
	for ( pnote = note_list; pnote != NULL; pnote = pnote->next )
	{
	    if ( is_note_to( ch, pnote ) && ( vnum++ == anum || fAll ) )
	    {
		sprintf( buf, "[%3d] %s: %s\r\n%s\r\nTo: %s\r\n",
		    vnum - 1,
		    pnote->sender,
		    pnote->subject,
		    pnote->date,
		    pnote->to_list
		    );
		send_to_char( buf, ch );
		send_to_char( pnote->text, ch );
		return;
	    }
	}

	send_to_char( "No such note.\r\n", ch );
	return;
    }

    if ( !str_cmp( arg, "+" ) )
    {
	note_attach( ch );
	strcpy( buf, ch->pnote->text );
	if ( strlen(buf) + strlen(argument) >= MAX_STRING_LENGTH - 4 )
	{
	    send_to_char( "Note too long.\r\n", ch );
	    return;
	}

	strcat( buf, argument );
	strcat( buf, "\r\n" );
	free_string( ch->pnote->text );
	ch->pnote->text = str_dup( buf );
	send_to_char( "Ok.\r\n", ch );
	return;
    }

    if ( !str_cmp( arg, "subject" ) )
    {
	note_attach( ch );
	free_string( ch->pnote->subject );
	ch->pnote->subject = str_dup( argument );
	send_to_char( "Ok.\r\n", ch );
	return;
    }

    if ( !str_cmp( arg, "to" ) )
    {
	note_attach( ch );
	free_string( ch->pnote->to_list );
	ch->pnote->to_list = str_dup( argument );
	send_to_char( "Ok.\r\n", ch );
	return;
    }

    if ( !str_cmp( arg, "clear" ) )
    {
	if ( ch->pnote != NULL )
	{
	    free_string( ch->pnote->text );
	    free_string( ch->pnote->subject );
	    free_string( ch->pnote->to_list );
	    free_string( ch->pnote->date );
	    free_string( ch->pnote->sender );
	    ch->pnote->next	= note_free;
	    note_free		= ch->pnote;
	    ch->pnote		= NULL;
	}

	send_to_char( "Ok.\r\n", ch );
	return;
    }

    if ( !str_cmp( arg, "show" ) )
    {
	if ( ch->pnote == NULL )
	{
	    send_to_char( "You have no note in progress.\r\n", ch );
	    return;
	}

	sprintf( buf, "%s: %s\r\nTo: %s\r\n",
	    ch->pnote->sender,
	    ch->pnote->subject,
	    ch->pnote->to_list
	    );
	send_to_char( buf, ch );
	send_to_char( ch->pnote->text, ch );
	return;
    }

    if ( !str_cmp( arg, "post" ) )
    {
	FILE *fp;
	char *strtime;

	if ( ch->pnote == NULL )
	{
	    send_to_char( "You have no note in progress.\r\n", ch );
	    return;
	}

	ch->pnote->next			= NULL;
	strtime				= ctime( &current_time );
	strtime[strlen(strtime)-1]	= '\0';
	ch->pnote->date			= str_dup( strtime );

	if ( note_list == NULL )
	{
	    note_list	= ch->pnote;
	}
	else
	{
	    for ( pnote = note_list; pnote->next != NULL; pnote = pnote->next )
		;
	    pnote->next	= ch->pnote;
	}
	pnote		= ch->pnote;
	ch->pnote	= NULL;

	fclose( fpReserve );
	if ( ( fp = fopen( NOTE_FILE, "a" ) ) == NULL )
	{
	    perror( NOTE_FILE );
	}
	else
	{
	    fprintf( fp, "Sender  %s~\nDate    %s~\nTo      %s~\nSubject %s~\nText\n%s~\n\n",
		pnote->sender,
		pnote->date,
		pnote->to_list,
		pnote->subject,
		pnote->text
		);
	    fclose( fp );
	}
	fpReserve = fopen( NULL_FILE, "r" );

	send_to_char( "Ok.\r\n", ch );
	return;
    }

    if ( !str_cmp( arg, "remove" ) )
    {
	if ( !is_number( argument ) )
	{
	    send_to_char( "Note remove which number?\r\n", ch );
	    return;
	}

	anum = atoi( argument );
	vnum = 0;
	for ( pnote = note_list; pnote != NULL; pnote = pnote->next )
	{
	    if ( is_note_to( ch, pnote ) && vnum++ == anum )
	    {
		note_remove( ch, pnote );
		send_to_char( "Ok.\r\n", ch );
		return;
	    }
	}

	send_to_char( "No such note.\r\n", ch );
	return;
    }

    send_to_char( "Huh?  Type 'help note' for usage.\r\n", ch );
    return;
}



/*
 * Generic channel function.
 */
void talk_channel( CHAR_DATA *ch, char *argument, int channel, const char *verb )
{
    char buf[MAX_STRING_LENGTH];
    DESCRIPTOR_DATA *d;
    int position;

    if ( argument[0] == '\0' )
    {
	sprintf( buf, "%s what?\r\n", verb );
	buf[0] = UPPER(buf[0]);
	return;
    }

    if ( !IS_NPC(ch) && IS_SET(ch->act, PLR_SILENCE) )
    {
	sprintf( buf, "You can't %s.\r\n", verb );
	send_to_char( buf, ch );
	return;
    }

    REMOVE_BIT(ch->deaf, channel);

    switch ( channel )
    {
    default:
	sprintf( buf, "{MYou %s:{x \"%s\"\r\n", verb, argument );
	send_to_char( buf, ch );
	sprintf( buf, "{M$n %ss:{x \"$t\"",     verb );
	break;
	
	case CHANNEL_CHAT:
	sprintf( buf, "{MYou %s:{x \"%s\"\r\n", verb, argument );
	send_to_char( buf, ch );
	sprintf( buf, "{M$n %ss:{x \"$t\"",     verb );
	break;
	
    case CHANNEL_IMMTALK:
	printf_to_char( ch, "{M%s{x: %s.\r\n", ch->name, argument );
	break;
    }

    for ( d = descriptor_list; d != NULL; d = d->next )
    {
	CHAR_DATA *och;
	CHAR_DATA *vch;

	och = d->original ? d->original : d->character;
	vch = d->character;

	if ( d->connected == CON_PLAYING
	&&   vch != ch
	&&  !IS_SET(och->deaf, channel) )
	{
	    if ( channel == CHANNEL_IMMTALK && !IS_HERO(och) )
		continue;
	    if ( channel == CHANNEL_YELL
	    &&   vch->in_room->area != ch->in_room->area )
		continue;

	    position		= vch->position;
	    if ( channel != CHANNEL_SHOUT && channel != CHANNEL_YELL )
		vch->position	= POS_STANDING;
	    act( buf, ch, argument, vch, TO_VICT );
	    vch->position	= position;
	}
    }

    return;
}



void do_auction( CHAR_DATA *ch, char *argument )
{
    talk_channel( ch, argument, CHANNEL_AUCTION, "auction" );
    return;
}



void do_chat( CHAR_DATA *ch, char *argument )
{
    talk_channel( ch, argument, CHANNEL_CHAT, "chat" );
    return;
}



/*
 * Alander's new channels.
 */
void do_music( CHAR_DATA *ch, char *argument )
{
    talk_channel( ch, argument, CHANNEL_MUSIC, "music" );
    return;
}



void do_question( CHAR_DATA *ch, char *argument )
{
    talk_channel( ch, argument, CHANNEL_QUESTION, "question" );
    return;
}



void do_answer( CHAR_DATA *ch, char *argument )
{
    talk_channel( ch, argument, CHANNEL_QUESTION, "answer" );
    return;
}



void do_shout( CHAR_DATA *ch, char *argument )
{
    talk_channel( ch, argument, CHANNEL_SHOUT, "shout" );
    return;
}



void do_yell( CHAR_DATA *ch, char *argument )
{
    talk_channel( ch, argument, CHANNEL_YELL, "yell" );
    return;
}



void do_immtalk( CHAR_DATA *ch, char *argument )
{
    talk_channel( ch, argument, CHANNEL_IMMTALK, "immtalk" );
    return;
}


void do_say( CHAR_DATA *ch, char *argument )
{
    if ( argument[0] == '\0' )
    {
	send_to_char( "Say what?\r\n", ch );
	return;
    }
	
    act( "{g$c says,{x \"$T\"", ch, NULL, argument, TO_ROOM );
    act( "{gYou say,{x \"$T\"", ch, NULL, argument, TO_CHAR );
    return;
}



void do_tell( CHAR_DATA *ch, char *argument )
{
    char arg[MAX_INPUT_LENGTH];
    CHAR_DATA *victim;

    if ( !IS_NPC(ch) && IS_SET(ch->act, PLR_SILENCE) )
    {
	send_to_char( "Your lips refuse to obey.\r\n", ch );
	return;
    }

    argument = one_argument( argument, arg );

    if ( arg[0] == '\0' || argument[0] == '\0' )
    {
	send_to_char( "Whisper to whom what?\r\n", ch );
	return;
    }
	

    if ( ( victim = get_char_room( ch, arg ) ) == NULL )
    {
	send_to_char( "Whisper to whom?\n\r", ch );
	return;
    }
	

    if ( !IS_IMMORTAL(ch) && !IS_AWAKE(victim) )
    {
	act( "Being asleeep, $N cannot hear your whispered words.", ch, 0, victim, TO_CHAR );
	return;
    }

    act( "{bYou whisper to $N,{x \"$t\"", ch, argument, victim, TO_CHAR );
    act( "{b$n whispers to you,{x \"$t\"", ch, argument, victim, TO_VICT );
    victim->reply	= ch;

    return;
}



void do_reply( CHAR_DATA *ch, char *argument )
{
    CHAR_DATA *victim;
    int position;

    if ( !IS_NPC(ch) && IS_SET(ch->act, PLR_SILENCE) )
    {
	send_to_char( "Your message didn't get through.\r\n", ch );
	return;
    }

    if ( ( victim = ch->reply ) == NULL )
    {
	send_to_char( "They aren't here.\r\n", ch );
	return;
    }

    if ( !IS_IMMORTAL(ch) && !IS_AWAKE(victim) )
    {
	act( "$E can't hear you.", ch, 0, victim, TO_CHAR );
	return;
    }

    act( "You tell $N, \"$t\"", ch, argument, victim, TO_CHAR );
    position		= victim->position;
    victim->position	= POS_STANDING;
    act( "$n tells you, \"$t\"", ch, argument, victim, TO_VICT );
    victim->position	= position;
    victim->reply	= ch;

    return;
}



void do_emote( CHAR_DATA *ch, char *argument )
{
    if ( !IS_NPC(ch) && IS_SET(ch->act, PLR_NO_EMOTE) )
    {
	send_to_char( "You can't use this command right now.\r\n", ch );
	return;
    }

    if ( argument[0] == '\0' )
    {
	send_to_char( "Act out what?\r\n", ch );
	return;
    }
	
    act( "($n $T)", ch, NULL, argument, TO_ROOM );
    act( "($n $T)", ch, NULL, argument, TO_CHAR );
    return;
}






void do_pose( CHAR_DATA *ch, char *argument )
{

    return;
}



// all one file

void do_bug( CHAR_DATA *ch, char *argument )
{
    append_file( ch, BUG_FILE, argument );
	
    printf_to_char( ch, "Thank you for reporting the following issue:\r\n%s\r\n", argument );
    return;
}



void do_idea( CHAR_DATA *ch, char *argument )
{
    append_file( ch, BUG_FILE, argument );
    send_to_char( "Ok.  Thanks.\r\n", ch );
    return;
}



void do_typo( CHAR_DATA *ch, char *argument )
{
    append_file( ch, BUG_FILE, argument );
    send_to_char( "Ok.  Thanks.\r\n", ch );
    return;
}



void do_rent( CHAR_DATA *ch, char *argument )
{
    send_to_char( "You cannot rent anything here.\r\n", ch );
    return;
}


void do_exi( CHAR_DATA *ch, char *argument )
{
    send_to_char( "Type EXIT or QUIT completely to leave the adventure.\r\n", ch );
    return;
}

void do_exit( CHAR_DATA *ch, char *argument )
{
    do_quit( ch, "" );
    return;
}




void do_qui( CHAR_DATA *ch, char *argument )
{
    send_to_char( "Type QUIT or EXIT completely to leave the adventure.\r\n", ch );
    return;
}


void do_quit( CHAR_DATA *ch, char *argument )
{
    DESCRIPTOR_DATA *d;
	DESCRIPTOR_DATA *d_next;
	static char name[MSL];

    if ( IS_NPC(ch) )
	return;

    //send_to_char("You quit.\r\n",ch );
    act( "$n just left.", ch, NULL, NULL, TO_ROOM );
    sprintf( log_buf, "%s has quit.", ch->name );
    log_string( log_buf );

// item duping bug fix below

    /*
     * After extract_char the ch is no longer valid!
     */

    save_char_obj( ch );
    strcpy(name, ch->name);
    d = ch->desc;
    extract_char( ch, TRUE );
	
    if ( d != NULL )
        close_socket( d );

    for (d = descriptor_list; d != NULL; d = d_next)
    {
        CHAR_DATA *tch;

        d_next = d->next;
        tch = d->original ? d->original : d->character;
        if (tch && !strcmp(tch->name,name) && ch != tch)
        {
            extract_char(tch,TRUE);
            close_socket(d);
        }
    }
    return;
}



void do_save( CHAR_DATA *ch, char *argument )
{
    if ( IS_NPC(ch) )
	return;

    save_char_obj( ch );
    send_to_char( "Ok.\r\n", ch );
    return;
}



void do_follow( CHAR_DATA *ch, char *argument )
{
    char arg[MAX_INPUT_LENGTH];
    CHAR_DATA *victim;

    one_argument( argument, arg );

    if ( arg[0] == '\0' )
    {
	send_to_char( "Join and follow whom?\r\n", ch );
	return;
    }

    if ( ( victim = get_char_room( ch, arg ) ) == NULL )
    {
	send_to_char( "They aren't here.\r\n", ch );
	return;
    }

    if ( IS_AFFECTED(ch, AFF_CHARM) && ch->master != NULL )
    {
	act( "But you'd rather follow $N!", ch, NULL, ch->master, TO_CHAR );
	return;
    }

    if ( victim == ch )
    {
	if ( ch->master == NULL )
	{
	    send_to_char( "You already follow yourself.\r\n", ch );
	    return;
	}
	stop_follower( ch );
	return;
    }

    if ( ch->master != NULL )
	stop_follower( ch );

    add_follower( ch, victim );
    return;
}



void add_follower( CHAR_DATA *ch, CHAR_DATA *master )
{
    if ( ch->master != NULL )
    {
	bug( "Add_follower: non-null master.", 0 );
	return;
    }

    ch->master        = master;
    ch->leader        = NULL;

    if ( can_see( master, ch ) )
	act( "$n now follows you.", ch, NULL, master, TO_VICT );

    act( "You now follow $N.",  ch, NULL, master, TO_CHAR );

    return;
}



void stop_follower( CHAR_DATA *ch )
{
    if ( ch->master == NULL )
    {
	bug( "Stop_follower: null master.", 0 );
	return;
    }

	/*
    if ( IS_AFFECTED(ch, AFF_CHARM) )
    {
	REMOVE_BIT( ch->affected_by, AFF_CHARM );
	affect_strip( ch, gsn_charm_person );
    }
	*/

    if ( can_see( ch->master, ch ) )
	act( "$n stops following you.",     ch, NULL, ch->master, TO_VICT    );
    act( "You stop following $N.",      ch, NULL, ch->master, TO_CHAR    );

    ch->master = NULL;
    ch->leader = NULL;
    return;
}



void die_follower( CHAR_DATA *ch )
{
    CHAR_DATA *fch;

    if ( ch->master != NULL )
	stop_follower( ch );

    ch->leader = NULL;

    for ( fch = char_list; fch != NULL; fch = fch->next )
    {
	if ( fch->master == ch )
	    stop_follower( fch );
	if ( fch->leader == ch )
	    fch->leader = fch;
    }

    return;
}



void do_order( CHAR_DATA *ch, char *argument )
{
    char arg[MAX_INPUT_LENGTH];
    CHAR_DATA *victim;
    CHAR_DATA *och;
    CHAR_DATA *och_next;
    bool found;
    bool fAll;

    argument = one_argument( argument, arg );

    if ( arg[0] == '\0' || argument[0] == '\0' )
    {
	send_to_char( "Tell whom to do what?\r\n", ch );
	return;
    }

    if ( IS_AFFECTED( ch, AFF_CHARM ) )
    {
	send_to_char( "You feel like taking, not giving, orders.\r\n", ch );
	return;
    }

    if ( !str_cmp( arg, "all" ) )
    {
	fAll   = TRUE;
	victim = NULL;
    }
    else
    {
	fAll   = FALSE;
	if ( ( victim = get_char_room( ch, arg ) ) == NULL )
	{
	    send_to_char( "They aren't here.\r\n", ch );
	    return;
	}

	if ( victim == ch )
	{
	    send_to_char( "Tell yourself?\r\n", ch );
	    return;
	}

	if ( !IS_AFFECTED(victim, AFF_CHARM) || victim->master != ch )
	{
	    send_to_char( "They are not likely to listen.\r\n", ch );
	    return;
	}
    }

    found = FALSE;
    for ( och = ch->in_room->people; och != NULL; och = och_next )
    {
	och_next = och->next_in_room;

	if ( IS_AFFECTED(och, AFF_CHARM)
	&&   och->master == ch
	&& ( fAll || och == victim ) )
	{
	    found = TRUE;
	    act( "$n orders you to \"$t\"", ch, argument, och, TO_VICT );
	    interpret( och, argument );
	}
    }

    if ( found )
	send_to_char( "Ok.\r\n", ch );
    else
	send_to_char( "You have no followers here.\r\n", ch );
    return;
}



void do_group( CHAR_DATA *ch, char *argument )
{
	//
}



/*
 * 'Split' originally by Gnort, God of Chaos.
 */
void do_split( CHAR_DATA *ch, char *argument )
{
    char buf[MAX_STRING_LENGTH];
    char arg[MAX_INPUT_LENGTH];
    CHAR_DATA *gch;
    int members;
    int amount;
    int share;
    int extra;

    one_argument( argument, arg );

    if ( arg[0] == '\0' )
    {
	send_to_char( "Share how much?\r\n", ch );
	return;
    }

    amount = atoi( arg );

    if ( amount < 2 )
    {
	send_to_char( "Sharing such an amount isn't sharing at all.\r\n", ch );
	return;
    }

    if ( ch->gold < amount )
    {
	send_to_char( "You don't have that much silver.\r\n", ch );
	return;
    }

    members = 0;
    for ( gch = ch->in_room->people; gch != NULL; gch = gch->next_in_room )
    {
	if ( is_same_group( gch, ch ) )
	    members++;
    }

    if ( members < 2 )
    {
	send_to_char( "Just keep it all.\r\n", ch );
	return;
    }

    share = amount / members;
    extra = amount % members;

    if ( share == 0 )
    {
	send_to_char( "There is not enough to share.\r\n", ch );
	return;
    }

    ch->gold -= amount;
    ch->gold += share + extra;

    sprintf( buf,
	"You split %d silver coins.  Your share is %d silver coins.\r\n",
	amount, share + extra );
    send_to_char( buf, ch );

    sprintf( buf, "$n splits %d silver coins.  Your share is %d silver coins.",
	amount, share );

    for ( gch = ch->in_room->people; gch != NULL; gch = gch->next_in_room )
    {
	if ( gch != ch && is_same_group( gch, ch ) )
	{
	    act( buf, ch, NULL, gch, TO_VICT );
	    gch->gold += share;
	}
    }

    return;
}



void do_gtell( CHAR_DATA *ch, char *argument )
{
    char buf[MAX_STRING_LENGTH];
    CHAR_DATA *gch;

    if ( argument[0] == '\0' )
    {
	send_to_char( "Tell your group what?\r\n", ch );
	return;
    }

    if ( IS_SET( ch->act, PLR_NO_TELL ) )
    {
	send_to_char( "Your message didn't get through!\r\n", ch );
	return;
    }

    /*
     * Note use of send_to_char, so gtell works on sleepers.
     */
    sprintf( buf, "%s tells the group '%s'.\r\n", ch->name, argument );
    for ( gch = char_list; gch != NULL; gch = gch->next )
    {
	if ( is_same_group( gch, ch ) )
	    send_to_char( buf, gch );
    }

    return;
}



/*
 * It is very important that this be an equivalence relation:
 * (1) A ~ A
 * (2) if A ~ B then B ~ A
 * (3) if A ~ B  and B ~ C, then A ~ C
 */
bool is_same_group( CHAR_DATA *ach, CHAR_DATA *bch )
{
    if ( ach->leader != NULL ) ach = ach->leader;
    if ( bch->leader != NULL ) bch = bch->leader;
    return ach == bch;
}
/***************************************************************************
 *  Original Diku Mud copyright (C) 1990, 1991 by Sebastian Hammer,        *
 *  Michael Seifert, Hans Henrik St{rfeldt, Tom Madsen, and Katja Nyboe.   *
 *                                                                         *
 *  Merc Diku Mud improvments copyright (C) 1992, 1993 by Michael          *
 *  Chastain, Michael Quan, and Mitchell Tse.                              *
 *                                                                         *
 *  In order to use any part of this Merc Diku Mud, you must comply with   *
 *  both the original Diku license in 'license.doc' as well the Merc       *
 *  license in 'license.txt'.  In particular, you may not remove either of *
 *  these copyright notices.                                               *
 *                                                                         *
 *  Much time and thought has gone into this software and you are          *
 *  benefitting.  We hope that you share your changes too.  What goes      *
 *  around, comes around.                                                  *
 ***************************************************************************/

#if defined(macintosh)
#include <types.h>
#else
#include <sys/types.h>
#endif
#include <ctype.h>
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <time.h>
#include "merc.h"

/*
#define SECT_INSIDE		      0
#define SECT_CITY		      1
#define SECT_FIELD		      2
#define SECT_FOREST		      3
#define SECT_HILLS		      4
#define SECT_MOUNTAIN		      5
#define SECT_WATER_SWIM		      6
#define SECT_WATER_NOSWIM	      7
#define SECT_SWAMP	      	8
#define SECT_AIR		      9
#define SECT_DESERT		     10
#define SECT_DUNGEON		 11
#define SECT_CAVES			 12
#define SECT_PLAINS			 13
#define SECT_RUINS			 14
#define SECT_MAX		     15
*/

const char *sector_type_names[SECT_MAX]=

    {"a room", "a street", "a field", "a forest", "some hills", "some mountains", "some water", 
     "some deep water", "a swamp", "some clouds", "a desert", "a dungeon", "a cave", "a flat plain" };
	 
const char *random_descs[10]=

    {"This is random description #1.", 
	"This is random description #2.", 
	"This is random description #3.", 
	"This is random description #4.", 
	"This is random description #5.", 
	"This is random description #6.", 
	"This is random description #7.", 
	"This is random description #8.", 
	"This is random description #9.", 
	"This is random description #10."};



/*
 * Local functions.
 */
char *	format_obj_to_char	args( ( OBJ_DATA *obj, CHAR_DATA *ch,
				    bool fShort ) );
void	show_list_to_char	args( ( OBJ_DATA *list, CHAR_DATA *ch,
				    bool fShort, bool fShowNothing ) );
char *	show_char_to_char_0	args( ( CHAR_DATA *victim, CHAR_DATA *ch ) );
void	show_char_to_char_1	args( ( CHAR_DATA *victim, CHAR_DATA *ch ) );
void	show_char_to_char	args( ( CHAR_DATA *list, CHAR_DATA *ch ) );
bool	check_blind		args( ( CHAR_DATA *ch ) );



//fixme
short number_of_items_inside( OBJ_DATA *obj )
{
	return 1;
}


char *format_obj_to_char( OBJ_DATA *obj, CHAR_DATA *ch, bool fShort )
{
    static char buf[MAX_STRING_LENGTH];

    buf[0] = '\0';

    if ( fShort )
    {
	if ( obj->short_descr != NULL )
	    strcat( buf, obj->short_descr );
    }
    else
    {
	if ( obj->description != NULL )
	    strcat( buf, obj->description );
    }
	
	if( (obj->item_type == ITEM_FURNITURE || obj->item_type == ITEM_MERCHANT_TABLE) && obj->contains != NULL ) 
		strcat( buf, " with something on it" ); //fixme

    return buf;
}

char * format_str_len(char * string, int length, int align)
{
    char buf[MSL];
	char buf2[MSL];
    char *new_string;
    char *count_string;
    char temp;
    int count = 0, nCount = 0;
    int pos = 0;
 
    new_string = buf;
	count_string = buf2;
	strcpy(buf2, string);

	/* get count for alignment */

	while( *count_string && nCount != length )
    {
        temp = *count_string++;
 
		if (temp == '{' )
        {
			temp = *count_string++;
			if (temp == '{')
                nCount++;
            continue;
        }
        nCount++;
    }

	/* force alignment of text to the right */
	if(align == ALIGN_RIGHT)
	{
		count = (length - ++nCount);
		while(nCount++ <= length)
		{
	        buf[pos++] = ' ';
		}
	}

	/* force alignment of text to the center */
	if(align == ALIGN_CENTER)
	{
		nCount = (length - nCount) / 2;
		count = nCount;
		while(nCount-- > 0)
		{
	        buf[pos++] = ' ';
		}
	}

	/* add to buffer */

	while( *string && count != length )
    {
        temp = *string++;
		buf[pos++] = temp;

		if (temp == '{' )
        {
            buf[pos] = *string++;

			if (buf[pos] == '{')
                count++;

			pos++;
            continue;
        }
        count++;
    }

	/* pad remaining space with blanks */
	while (count++ < length)
        buf[pos++] = ' ';

	buf[pos] = '\0';

	return (new_string);
}

void printwrap(CHAR_DATA *ch, const char *s, int lineSize, const char *prefix) {
	const char *head = s;
	int pos, lastSpace;
	
	pos = lastSpace = 0;
	while(head[pos]!=0) 
	{
		int isLf = (head[pos]=='\n');
		if (isLf || pos==lineSize) 
		{
			if (isLf || lastSpace == 0) 
			{ 
			lastSpace = pos; 
			} // just cut it
			
			if (prefix!=NULL) 
			{ 
			printf_to_char(ch, "%s", prefix); 
			}
			
			while(*head!=0 && lastSpace-- > 0) 
			{ 
			printf_to_char(ch, "%c", *head++); 
			}
			
			printf_to_char(ch, "\r\n");
			
			if (isLf) 
			{ 
			head++; 
			} // jump the line feed
			
			while (*head!=0 && *head==' ') 
			{ 
			head++; 
			} // clear the leading space
			
			lastSpace = pos = 0;
		} 
		else 
		{ 
			if (head[pos]==' ') 
			{ 
			lastSpace = pos; 
			}
			pos++;
		}
	}
	printf_to_char(ch, "%s", head);
}

bool new_show_list_to_char( OBJ_DATA *list, CHAR_DATA *ch, bool fShort,
  bool fShowNothing, char *pre )
{
    OBJ_DATA *obj;
    int count;
    bool found = FALSE;
	static char buf[MSL];
	
	strcpy( buf, "" );
	
	strcat( buf, pre );

    if ( ch->desc == NULL )
	return FALSE;

    count = 0;
	
	for ( obj = list; obj != NULL; obj = obj->next_content )
    { 

	if ( obj->wear_loc == EQUIP_NONE && can_see_obj( ch, obj ) )
	{
        found = TRUE;
		count++;

	}
	
	}
	
	if ( fShowNothing && count == 0 )
	{
    	strcat( buf, "nothing." );
	}

    /*
     * Format the list of objects.
     */
    for ( obj = list; obj != NULL; obj = obj->next_content )
    { 
	if ( obj->wear_loc == EQUIP_NONE && can_see_obj( ch, obj ) )
	{
	    strcat( buf, format_obj_to_char( obj, ch, fShort ) );
		
		count--;
		
		if( count == 0 )
		strcat( buf, "." );
		else if( count == 1 )
		strcat( buf, " and " );
		else
		strcat( buf, ", " );
	}
    }
	
	strcat( buf, "\r\n" );
	
	if( found || fShowNothing )
	{
	if( !IS_NPC(ch) && IS_SET( ch->act, PLR_WRAP ) )
	printf_to_char( ch, "%s", buf ); 
	else
	send_to_char( buf, ch );
	}

    return found;
}

bool new2_show_list_to_char( OBJ_DATA *list, CHAR_DATA *ch, bool fShort,
  bool fShowNothing, char *pre )
{
    OBJ_DATA *obj;
    int count;
    bool found = FALSE;
	static char buf[MSL];
	
	strcpy( buf, "" );
	
	strcat( buf, pre );

    if ( ch->desc == NULL )
	return FALSE;

    count = 0;
	
	for ( obj = list; obj != NULL; obj = obj->next_content )
    { 

	if ( can_see_obj( ch, obj ) )
	{
        found = TRUE;
		count++;

	}
	
	}
	
	if ( fShowNothing && count == 0 )
	{
    	strcat( buf, "nothing." );
	}

    /*
     * Format the list of objects.
     */
    for ( obj = list; obj != NULL; obj = obj->next_content )
    { 
	if ( can_see_obj( ch, obj ) )
	{
	    strcat( buf, format_obj_to_char( obj, ch, fShort ) );
		
		count--;
		
		if( count == 0 )
		strcat( buf, "." );
		else if( count == 1 )
		strcat( buf, " and " );
		else
		strcat( buf, ", " );
	}
    }
	
	strcat( buf, "\r\n" );
	
	if( found || fShowNothing )
	{
	if( !IS_NPC(ch) && IS_SET( ch->act, PLR_WRAP ) )
	printf_to_char( ch, "%s", buf );
	else
	send_to_char( buf, ch );
	}

    return found;
}

void show_hands_to_char( CHAR_DATA *victim, CHAR_DATA *ch )
{
	OBJ_DATA *obj1 = get_eq_char( victim, EQUIP_RIGHTHAND );
	OBJ_DATA *obj2 = get_eq_char( victim, EQUIP_LEFTHAND );
	char buf[MSL];
	
	strcpy( buf, "" );
	
	if( victim != ch )
	{
	if( obj1 != NULL && obj2 != NULL )
	{
	sprintf(buf, 
	"%s is holding %s in %s right hand and %s in %s left hand.\r\n",
	victim->sex == 1 ? "He" : victim->sex == 2 ? "She" : "It",
	obj1->short_descr, 
	victim->sex == 1 ? "his" : victim->sex == 2 ? "her" : "its",
	obj2->short_descr,
	victim->sex == 1 ? "his" : victim->sex == 2 ? "her" : "its"	);
	}
	else if( obj1 != NULL && obj2 == NULL )
	{
	sprintf(buf, 
	"%s is holding %s in %s right hand.\r\n",
	victim->sex == 1 ? "He" : victim->sex == 2 ? "She" : "It",
	obj1->short_descr,
	victim->sex == 1 ? "his" : victim->sex == 2 ? "her" : "its"	);
	}
	else if( obj1 == NULL && obj2 != NULL )
	{
	sprintf(buf,  
	"%s is holding %s in %s left hand.\r\n",
	victim->sex == 1 ? "He" : victim->sex == 2 ? "She" : "It",
	obj2->short_descr,
	victim->sex == 1 ? "his" : victim->sex == 2 ? "her" : "its"	);
	}
	}
	else
	{
	if( obj1 != NULL && obj2 != NULL )
	{
	sprintf(buf,
	"You are holding %s in your right hand and %s in your left hand.\r\n",
	obj1->short_descr, obj2->short_descr );
	}
	else if( obj1 != NULL && obj2 == NULL )
	{
	sprintf(buf, 
	"You are holding %s in your right hand.\r\n",
	obj1->short_descr );
	}
	else if( obj1 == NULL && obj2 != NULL )
	{
	sprintf(buf, 
	"You are holding %s in your left hand.\r\n",
	obj2->short_descr );
	}
	}
	
	if( strlen(buf) > 2 )
	{
		if( !IS_NPC(ch) && IS_SET( ch->act, PLR_WRAP ) )
		printf_to_char( ch, "%s", buf );
	else
		send_to_char( buf, ch );
	}
	
	return;
}

bool show_eq_to_char( CHAR_DATA *victim, CHAR_DATA *ch, bool fShort,
  bool fShowNothing )
{
    char **prgpstrShow;
    //int *prgnShow;
    char *pstrShow;
    OBJ_DATA *obj;
    int nShow;
    int iShow;
    int count;
    bool found = FALSE;
	char buf[MSL];
	
	strcpy( buf, "" );

    if ( ch->desc == NULL )
	return FALSE;

    /*
     * Alloc space for output lines.
     */
    count = 0;
    for ( obj = victim->carrying; obj != NULL; obj = obj->next_content )
	count++;
    prgpstrShow	= (char **) alloc_mem( count * sizeof(char *) );
    //prgnShow    = (int *) alloc_mem( count * sizeof(int)    );
    nShow	= 0;
	
	show_hands_to_char( victim, ch );

	if( ch != victim )
    sprintf( buf, "%s is wearing ", victim->sex == 1 ? "He" : victim->sex == 2 ? "She" : "It" );
	else
	strcat( buf, "You are wearing " );

    /*
     * Format the list of objects.
     */
    for ( obj = victim->carrying; obj != NULL; obj = obj->next_content )
    { 
	if ( ( obj->wear_loc != EQUIP_NONE && obj->wear_loc != EQUIP_RIGHTHAND && obj->wear_loc != EQUIP_LEFTHAND )
	&& can_see_obj( ch, obj ) )
	{
            found = TRUE;

	    pstrShow = format_obj_to_char( obj, ch, fShort );

		prgpstrShow [nShow] = str_dup( pstrShow );
		//prgnShow    [nShow] = 1;
		nShow++;
	}
    }

    /*
     * Output the formatted list.
     */
    {

    for ( iShow = 0; iShow < nShow; iShow++ )
    {
	strcat( buf, prgpstrShow[iShow] );

    if ( iShow < nShow-2 )
    {
	strcat( buf, ", " );
    }
    else if ( iShow == nShow-2 )
    {
	strcat( buf, " and " );
    }
    else
    {
	strcat( buf, "." );
    }

	free_string( prgpstrShow[iShow] );
    }

    }

    if ( fShowNothing && nShow == 0 )
	{
    	strcat( buf, "nothing special." );
	}
	//output
	//	send_to_char( buf, ch );
	
	strcat( buf, "\r\n" );
	
	if( found || fShowNothing )
	{
	if( !IS_NPC(ch) && IS_SET(ch->act, PLR_WRAP ) )
	printf_to_char( ch, "%s", buf );
	else
	send_to_char( buf, ch );
	}
		
    /*
     * Clean up.
     */
    free_mem( prgpstrShow, count * sizeof(char *) );
    //free_mem( prgnShow,    count * sizeof(int)    );

    return found;
}



char *show_char_to_char_0( CHAR_DATA *victim, CHAR_DATA *ch )
{
    static char buf[MAX_STRING_LENGTH];
	bool smash = FALSE;

    buf[0] = '\0';
	
	strcat( buf, PERS( victim, ch ) );

	if ( victim->stun > 0   )
	{
		smash = TRUE;
		strcat( buf, " (stunned)"      );
	}
		
	if ( IS_SET( victim->affected_by, AFF_DEAD )  ) 
	{
		smash = TRUE;
		strcat( buf, " (dead)"      );
	}

	if( !smash )
	{
    switch ( victim->position )
    {
    case POS_PRONE:  strcat( buf, " (prone)" ); break;
    case POS_SITTING: strcat( buf, " (sitting)" );      break;
    case POS_KNEELING:  strcat( buf, " (kneeling)" );       break;
    default:
    case POS_STANDING: break;
    }
	}

    //send_to_char( buf, ch );
	return buf;
    //return;
}



char *get_hair( CHAR_DATA *ch )
{
	if(IS_NPC(ch))
		return "bugged";
	
	if( ch->pcdata->hair != NULL && strlen(ch->pcdata->hair) > 2)
		return ch->pcdata->hair;
	
	if(!str_cmp(ch->pcdata->hair, "(null)"))
	return "black";

	else
		return "bugged";
}


char *get_eyes( CHAR_DATA *ch )
{
	if(IS_NPC(ch))
		return "bugged";
	
	if( ch->pcdata->eyes != NULL && strlen(ch->pcdata->eyes) > 2)
		return ch->pcdata->eyes;
	
	if(!str_cmp(ch->pcdata->eyes, "(null)"))
	return "brown";

	else
		return "bugged";
}


char *get_skin( CHAR_DATA *ch )
{
	if(IS_NPC(ch))
		return "bugged";
	
	if( ch->pcdata->skin != NULL && strlen(ch->pcdata->skin) > 2)
		return ch->pcdata->skin;
	
	if(!str_cmp(ch->pcdata->skin, "(null)"))
	return "fair";

	else
		return "bugged";
}

char *pos_string( CHAR_DATA *ch )
{
	if( ch->position == POS_PRONE )
	return "lying down";
	
	else if( ch->position == POS_SITTING )
	return "sitting";
	
	else if( ch->position == POS_KNEELING )
	return "kneeling";
	
	else if( ch->position == POS_STANDING )
	return "standing";
	
	else
	return "**BUGGED**";

}


void show_char_to_char_1( CHAR_DATA *victim, CHAR_DATA *ch )
{
    
	/*
	if ( victim->description[0] != '\0' )
    {
	send_to_char( victim->description, ch );
    }
	*/
	
	if( IS_NPC( victim ) )
	{
		if( IS_SET(victim->act,ACT_AGGRESSIVE))
		{
		printf_to_char( ch, "You see a typical %s that is %s.\r\n", smash_article( victim->short_descr ),
			pos_string( victim ) );
		}
		else
		{
		if( strlen( victim->description ) > 2 )
		{
			if( !IS_NPC(ch) && IS_SET( ch->act, PLR_WRAP ) )
			printf_to_char( ch, "%s", victim->description );
			else
			send_to_char( victim->description, ch );  //typically town mobs will have descriptions
		}
		else if( strlen( victim->long_descr ) > 2 )  
		{
			if( !IS_NPC(ch) && IS_SET( ch->act, PLR_WRAP ) )
			printf_to_char( ch, "%s", victim->long_descr );
			else
			send_to_char( victim->long_descr, ch );  
		}
		}
	
	}
	else
	{
		printf_to_char( ch, "You see %s the %s.\r\n", victim->name, race_table[victim->race].name );
		//fixme wrap
		printf_to_char(ch, "%s appears to be in %s %d0's, has %s hair, %s eyes, and %s skin.\r\n", victim->sex == 1 ? "He" : victim->sex == 2 ? "She" : "It",
			victim->sex == 1 ? "his" : victim->sex == 2 ? "her" : "its", victim->pcdata->age/10,
				get_hair(victim), get_eyes(victim), get_skin(victim));
	}

	//fixme show health
	{
	char buf[MAX_STRING_LENGTH];
	bool found = FALSE;
	int iBody = 1;
	
	strcpy( buf, "" );
	
	while( iBody < MAX_BODY )
	{
		if( victim->body[iBody] > 0 && victim->body[iBody] < 4 )
		{
			strcat( buf, victim->sex == 1 ? "His " : victim->sex == 2 ? "Her " : "Its " );
			strcat( buf, body_name[iBody] );
			strcat( buf, " is " );
			
			if( victim->body[iBody] == 1 )
				strcat( buf, "lightly wounded.  ");
			else if( victim->body[iBody] == 2 )
				strcat( buf, "seriously wounded.  ");
			else if( victim->body[iBody] == 3 )
				strcat( buf, "critically wounded.  ");
			
			found = TRUE;
		}	
	iBody++;
	}
	
	iBody = 0;
	
	while( iBody < MAX_BODY )
	{
		if( victim->scars[iBody] > 0 && victim->scars[iBody] < 4 )
		{
			strcat( buf, victim->sex == 1 ? "His " : victim->sex == 2 ? "Her " : "Its " );
			strcat( buf, body_name[iBody] );
			strcat( buf, " is " );
			
			if( victim->scars[iBody] == 1 )
				strcat( buf, "faintly scarred.  ");
			else if( victim->scars[iBody] == 2 )
				strcat( buf, "moderately scarred.  ");
			else if( victim->scars[iBody] == 3 )
				strcat( buf, "heavily scarred.  ");
			
			found = TRUE;
		}	
	iBody++;
	}
	
	iBody = 0;
	
	while( iBody < MAX_BODY )
	{
		if( victim->bleed[iBody] > 0 && victim->bandage[iBody] < 1 )
		{
			strcat( buf, victim->sex == 1 ? "His " : victim->sex == 2 ? "Her " : "Its " );
			strcat( buf, body_name[iBody] );
			strcat( buf, " is " );
			
			if( victim->bleed[iBody] < 3 )
				strcat( buf, "bleeding lightly.  ");
			else if( victim->bleed[iBody] < 5 )
				strcat( buf, "bleeding.  ");
			else
				strcat( buf, "bleeding heavily.  ");
			
			found = TRUE;
		}	
	iBody++;
	}
	
	if( found )
	{
	if( !IS_NPC(ch) && IS_SET(ch->act, PLR_WRAP ) )
	printf_to_char( ch, "%s", buf );
	else
	{
	send_to_char( buf, ch );
	}
	}
	
	if( found )
	send_to_char( "\r\n", ch );
	
	if( !IS_NPC(ch) && !found )
	printf_to_char(ch, "%s is in good shape.\r\n", victim->sex == 1 ? "He" : victim->sex == 2 ? "She" : "It" );
	}

	show_eq_to_char( victim, ch, TRUE, TRUE );

    return;
}



void show_char_to_char( CHAR_DATA *list, CHAR_DATA *ch )
{
    CHAR_DATA *rch;
	static char buf[MSL];
	int people = 0;
	
	strcpy( buf, "" );
	
	for ( rch = list; rch != NULL; rch = rch->next_in_room )
    {
	if ( rch == ch )
	    continue;

	if ( !IS_NPC(rch)
	&&   IS_SET(rch->act, PLR_WIZINVIS)
	&&   get_trust( ch ) < get_trust( rch ) )
	    continue;

	if ( !IS_NPC(rch) && can_see( ch, rch ) != FALSE  )
	{
	    people++;
	}
    }
	
	if( people > 0 )
	{
	strcat( buf, "Also here: " );

    for ( rch = list; rch != NULL; rch = rch->next_in_room )
    {
	if ( rch == ch )
	    continue;

	if ( !IS_NPC(rch)
	&&   IS_SET(rch->act, PLR_WIZINVIS)
	&&   get_trust( ch ) < get_trust( rch ) )
	    continue;

	if ( !IS_NPC(rch) && can_see( ch, rch ) != FALSE  )
	{
	    strcat( buf, show_char_to_char_0( rch, ch ));
		
		people--;
		
		if( people > 0 )
		strcat( buf, ", " );
	}
    }
	
		strcat( buf, "\r\n" );
		
	    if( !IS_NPC(ch) && IS_SET(ch->act, PLR_WRAP ) )
		printf_to_char( ch, "%s", buf );
		else
		send_to_char( buf, ch );
	}
	
    return;
}

void replace_char_from_string(char from, char to, char *str)
{
    int i = 0;
    int len = strlen(str)+1;

    for(i=0; i<len; i++)
    {
        if(str[i] == from)
        {
            str[i] = to;
        }
    }
}


//fixme seasons, night/day, moon(s)********************

char *moon_phase( int day )
{
switch( day )
{
case 1: case 2: case 3: case 4: 
return "new moon"; break;

case 5: case 6: case 7: case 8: 
return "waxing crescent moon"; break;

case 9: case 10: case 11: case 12:
return "first quarter moon"; break;

case 13: case 14: case 15: case 16:
return "waxing gibbous moon"; break;

case 17: case 18: case 19:
return "full moon"; break;

case 20: case 21: case 22: case 23:
return "waxing gibbous moon"; break;

case 24: case 25: case 26: case 27: 
return "last quarter moon"; break;

default:
case 28: case 29: case 30: case 31:
return "waning crescent moon"; break;
}
}

char *generate_dynamic_description( ROOM_INDEX_DATA *room )
{
	int door;
	EXIT_DATA *pexit;
	static char buf[MSL];
	static char buf2[MSL];
	char *paths[10];
	int exits = 0;
	bool found = FALSE;
	bool sameSector = TRUE;
	extern char * const dir_name[];
	
	strcpy( buf, "" );
	
	strcpy( buf, room->area->desc[room->vnum%10] );
	
	// skipping 'out' directions no longer
	
	for ( door = 0; door < DIR_MAX; door++ )
    {
	if ( ( pexit = room->exit[door] ) != NULL
	&&   pexit->to_room != NULL )
	{
		paths[exits] = str_dup( dir_name[door] );
		exits++;
		found = TRUE;
		
		
		if( room->sector_type != pexit->to_room->sector_type )
			sameSector = FALSE;
	}
    }
	
	if( sameSector == TRUE && room->sector_type == SECT_FOREST )
	{
	switch( exits )
	{
	case 0: 	strcat( buf, "  The path has disappeared.  You are trapped in the forest.");  break;
	case 1:
	sprintf( buf2, "  A path leads %s, out of the thick forest undergrowth.", paths[0] );
		strcat( buf, buf2);  break;
	case 2:
		sprintf( buf2, "  A path winds its way through the thick forest, twisting %s and %s.", paths[1], paths[0]  );
		strcat( buf, buf2);  break;
	case 3:
		sprintf( buf2, "  Several paths fork here, leading in three directions." );
		strcat( buf, buf2);  break;
	case 4:
		strcat( buf, "  One path through the forest meets another, forming a crossroads in the woods.");  break;
	case 5: case 6: case 7: case 8: case 9: case 10: default:
	strcat( buf, "  From this forest clearing, there are many paths going in all directions of the Greywood.");  break;
	}
	}
	
	if( sameSector == TRUE && room->sector_type == SECT_SWAMP )
	{
	switch( exits )
	{
	case 0: 	strcat( buf, "  All around you is black corruption.  You are trapped in the swamp.");  break;
	case 1:
	sprintf( buf2, "  The thick, fetid vegetation and roots and the deep, dark water close in, allowing you to only go %s from this point.", paths[0] );
		strcat( buf, buf2);  break;
	case 2:
		sprintf( buf2, "   A trail leads through the bog, going %s and %s.", paths[1], paths[0] );
		strcat( buf, buf2);  break;
	case 3:
		sprintf( buf2, "  A trail splits in the marsh, leading %s, %s, and %s.", paths[2], paths[0], paths[1] );
		strcat( buf, buf2);  break;
	default:
	break;
	}
	}
	
		if( sameSector == TRUE && room->sector_type == SECT_CITY )
	{
	
	//fixme moon phase
	{
		time_t rawtime;
		struct tm * timeinfo;

		time ( &rawtime );
		timeinfo = localtime ( &rawtime );
	
	if( room->vnum%3 )
	{
	switch( timeinfo->tm_hour )
	{
	case 22:
	case 23:
	case 0: 
	case 1:
	case 2:
	case 3:
	case 4:
	strcat( buf, "  The streets are empty at this late hour."); break;
	
	case 5: case 6: case 7: 
	strcat( buf, "  The town is beginning to stir as early risers go to and fro in the streets."); break;
	
	case 8: case 9: case 10: 
	strcat( buf, "  The town is in full swing as the people go about their morning business."); break;
	
	case 11: case 12: case 13: 
	strcat( buf, "  The town is bustling with mid-day activity."); break;
	
	case 14: case 15: case 16: case 17: 
	strcat( buf, "  The town is boisterous as the townsfolk conduct their afternoon affairs."); break;
	
	case 18: case 19: 
	strcat( buf, "  It is quite late, and most of the respectable townsfolk have gone home by now."); break;
	
	case 20: case 21:
	strcat( buf, "  It is very late, and only a drunken reveler or two can be spotted walking the streets."); break;
	
	default:
	break;
	}
	}
	
	if( timeinfo->tm_hour < 4 && timeinfo->tm_hour > 17 /* && room->vnum%3 */ )
	{
switch( timeinfo->tm_mday )
{
case 1: case 2: case 3: case 4: 
strcat(buf, "  The sky is dark as it is the time of the new moon."); break;

case 5: case 6: case 7: case 8: 
strcat(buf, "  The waxing crescent moon barely illuminates the street."); break;

case 9: case 10: case 11: case 12:
strcat(buf, "  The first quarter moon provides some dim light to the city."); break;

case 13: case 14: case 15: case 16:
strcat(buf, "  The waxing gibbous moon gleams off the rooftops."); break;

case 17: case 18: case 19:
strcat(buf, "  A rich, luminous full moon hangs over the city, casting silvery light everywhere."); break;

case 20: case 21: case 22: case 23:
strcat(buf, "  The waning gibbous moon gleams off the rooftops."); break;

case 24: case 25: case 26: case 27:
strcat(buf, "  The last quarter moon provides some dim light to the city."); break;

default:
case 28: case 29: case 30: case 31:
strcat(buf, "  The waning crescent moon barely illuminates the street."); break;
}
	}
	
	switch( exits )
	{
	case 0: 
		strcat( buf, "  The buildings have closed in upon you.  You are trapped.");  break;
	case 1:
	sprintf( buf2, "  This is a dead end, with only the way %s leading out.", paths[0] );
		strcat( buf, buf2);  break;
	case 2:
		sprintf( buf2, "  The street allows for traffic to go only %s or %s.", paths[1], paths[0] );
		strcat( buf, buf2);  break;
	case 3:
	strcat( buf, "  A main street lies here, with a thoroughfare for lighter traffic going another way."); break;
	case 4:
		strcat( buf, "  The streets meet in a four-way intersection here."); break;
	case 5: case 6: case 7: case 8: case 9: case 10: default:
	strcat( buf, "  Streets at various angles split from this intersection."); break;
	}

	//fixme night -- street lamps
	}
	}
	
	if( sameSector == TRUE && room->sector_type == SECT_DUNGEON )
	{
	switch( exits )
	{
	case 0:	strcat( buf, "  The walls have closed in on you.  You are trapped.");  break;
	case 1:
	sprintf( buf2, "  This is a dead end, with %s being the only way.", paths[0] );
		strcat( buf, buf2);  break;
	case 2:
		sprintf( buf2, "  This passageway continues %s and %s.", paths[1], paths[0] );
		strcat( buf, buf2);  break;
	case 3:
		sprintf( buf2, "  The corridors fork, leading %s, %s, and %s.", paths[2], paths[0], paths[1] );
		strcat( buf, buf2);  break;
	case 4:
	case 5: case 6: case 7: case 8: case 9: case 10: default:
 break;
	}
	}
	
	if( sameSector == TRUE && room->sector_type == SECT_RUINS )
	{
	switch( exits )
	{
	case 0: 	strcat( buf, "  The rubble has buried you.  You are trapped.");  break;
	case 1:
	sprintf( buf2, "  This is a dead end among the heaps of ruins, and going back %s is the only option.", paths[0] );
		strcat( buf, buf2);  break;
	case 2:
		sprintf( buf2, "  The ruins close in, allowing you to only continues %s and %s.", paths[1], paths[0] );
		strcat( buf, buf2);  break;
	case 3:
		sprintf( buf2, "  The ruins lead %s, %s, and %s.", paths[2], paths[0], paths[1] );
		strcat( buf, buf2);  break;
	case 4:
	case 5: case 6: case 7: case 8: case 9: case 10: default:
 break;
	}
	}
	
	if( sameSector == TRUE && room->sector_type == SECT_FIELD )
	{
	switch( exits )
	{
	case 0: 	strcat( buf, "  There is nowhere to go.");  break;
	case 1:
	sprintf( buf2, "  Going back %s is the only option.", paths[0] );
		strcat( buf, buf2);  break;
	case 2:
		sprintf( buf2, "  You can continue %s and %s.", paths[1], paths[0] );
		strcat( buf, buf2);  break;
	case 3:
		sprintf( buf2, "  The way leads %s, %s, and %s.", paths[2], paths[0], paths[1] );
		strcat( buf, buf2);  break;
	case 4:
	case 5: case 6: case 7: case 8: case 9: case 10: default:
 break;
	}
	}
	
	if( sameSector == TRUE && room->sector_type == SECT_CAVES )
	{
	switch( exits )
	{
	case 0: 	strcat( buf, "  You cannot find the tunnel from which you came.  You are trapped.");  break;
	case 1:
	sprintf( buf2, "  The tunnel comes to an abrupt stop.  You can only go back %s.", paths[0] );
		strcat( buf, buf2);  break;
	case 2:
		sprintf( buf2, "  This tunnel goes %s and %s.", paths[1], paths[0] );
		strcat( buf, buf2);  break;
	case 3:
		sprintf( buf2, "  Three tunnels lead from here, going %s, %s, and %s.", paths[2], paths[0], paths[1] );
		strcat( buf, buf2);  break;
	case 4:
		sprintf( buf2, "  Four different tunnels branch outward, leaving %s, %s, %s, and %s as possible directions to go.", paths[3], paths[2], paths[0], paths[1] );
		strcat( buf, buf2);  break;
	case 5: case 6: case 7: case 8: case 9: case 10: default:
 break;
	}
	}
	
	if( sameSector == TRUE && room->sector_type == SECT_PLAINS )
	{
	switch( exits )
	{
	case 0: 	strcat( buf, "  The grasses close in on you.  You are trapped.");  break;
	case 1:
	sprintf( buf2, "  The thick grasses converge, leaving only %s as a way out.", paths[0] );
		strcat( buf, buf2);  break;
	case 2:
		sprintf( buf2, "  A path cuts through the thick grasses, heading %s and %s.", paths[1], paths[0] );
		strcat( buf, buf2);  break;
	case 3:
		sprintf( buf2, "  A fork in paths cut through the thick grasses allows travel %s, %s, and %s.", paths[2], paths[0], paths[1] );
		strcat( buf, buf2);  break;
	case 4:
		sprintf( buf2, "  The flat grassland allows you to head %s, %s, %s, and %s.", paths[3], paths[2], paths[0], paths[1] );
		strcat( buf, buf2);  break;
	case 5: case 6: case 7: case 8: case 9: case 10: default:
 break;
	}
	}
	
	if( sameSector == TRUE && room->sector_type == SECT_DESERT )
	{
	switch( exits )
	{
	case 0: 	strcat( buf, "  The dunes rise high above you as sand scatters and shifts around you.  You are trapped.");  break;
	case 1:
	sprintf( buf2, "  The desert comes to an abrupt end in a steep rocky outcropping, allowing only egress %s.", paths[0] );
		strcat( buf, buf2);  break;
	case 2:
		sprintf( buf2, "  A path cuts through the sand dunes, going %s and %s.", paths[1], paths[0] );
		strcat( buf, buf2);  break;
	case 3:
		sprintf( buf2, "  You are at the top of a sand dune, with paths leading %s, %s, and %s.", paths[2], paths[0], paths[1] );
		strcat( buf, buf2);  break;
	case 4:
		sprintf( buf2, "  The rolling desert sands continue %s, %s, %s, and %s.", paths[3], paths[2], paths[0], paths[1] );
		strcat( buf, buf2);  break;
	case 5: case 6: case 7: case 8: case 9: case 10: default:
 break;
	}
	}
	
	if( sameSector == TRUE && room->sector_type == SECT_INSIDE )
	{
	switch( exits )
	{
	case 0:
		strcat( buf, "  There are no exits.  You are trapped inside.");  
	break;
	
	case 1:
	sprintf( buf2, "  %s is only way out of this room.", capitalize(paths[0]) );
		strcat( buf, buf2);  break;
		
		case 2:
		sprintf( buf2, "  You can go %s or %s from here.", paths[1], paths[0] );
		strcat( buf, buf2);  break;
	default:
	break;
	}
	}
	
	if( sameSector == FALSE )
	{
	if( found )
	{
	if( strlen(buf) > 3 )
	strcat( buf, "  There is ");
	else
	strcat( buf, "There is ");
	}
	
    for ( door = 0; door <= 9; door++ )
    {
	if ( ( pexit = room->exit[door] ) != NULL
	&&   pexit->to_room != NULL )
	{
		strcat( buf, sector_type_names[pexit->to_room->sector_type] );
	
		strcat( buf, " to the " );
	
		strcat( buf, dir_name[door] );
		
		exits--;
		
		if( exits == 0 )
		strcat( buf, "." );
		else if( exits == 1 )
		strcat( buf, " and " );
		else
		strcat( buf, ", " );
	}
    }
	}
	
	strcat( buf, "  ");
	
	return buf;
}


void show_combined_to_char( CHAR_DATA *char_list, OBJ_DATA *obj_list, CHAR_DATA *ch )
{
    CHAR_DATA *rch;
	OBJ_DATA *obj;
	static char buf[MSL];
	int number = 0;
	
	strcpy( buf, "" );
	
	if( !IS_NPC(ch) && !IS_SET( ch->act, PLR_BRIEF ) )
	{
		
	if( strlen(ch->in_room->description) > 5 )
	{
	
	if( IS_SET( ch->in_room->room_flags, ROOM_DYNAMIC ) )
	{
		sprintf(buf, "%s %s", ch->in_room->description, generate_dynamic_description( ch->in_room ) );
		replace_char_from_string( '\r', ' ', buf );
		replace_char_from_string( '\n', ' ', buf );	
	}
	else
	{
		strcpy( buf, ch->in_room->description);
		replace_char_from_string( '\r', ' ', buf );
		replace_char_from_string( '\n', ' ', buf );	
	}

	}
	else
	{
	strcpy( buf, generate_dynamic_description( ch->in_room ) );
	}
	}
	else
	{
	strcpy( buf, "" );
	}
	
	for ( rch = char_list; rch != NULL; rch = rch->next_in_room )
    {

	if (  rch != NULL && rch != ch && can_see( ch, rch ) && IS_NPC( rch ) )
	{
	    number++;
	}
	else
		continue;
    }
	
	for ( obj = obj_list; obj != NULL; obj = obj->next_content )
    { 
	if ( obj->wear_loc == EQUIP_NONE && can_see_obj( ch, obj ) )
	{
		number++;
	}
	}
	
	if( number > 0 )
	{
	if( !IS_NPC(ch) && IS_SET(ch->act, PLR_WRAP ) )
	strcat( buf, "  You also see " );
	else
	strcat( buf, "You also see " );
	}

    for ( obj = obj_list; obj != NULL; obj = obj->next_content )
    { 
	if ( obj->wear_loc == EQUIP_NONE && can_see_obj( ch, obj ) )
	{
	    strcat( buf, format_obj_to_char( obj, ch, TRUE ) );
		
		number--;
		
		if( number == 0 )
		strcat( buf, "." );
		else if( number == 1 )
		strcat( buf, " and " );
		else
		strcat( buf, ", " );
	}
    }

    for ( rch = char_list; rch != NULL; rch = rch->next_in_room )
    {
	if ( rch == ch )
	    continue;

	if ( can_see( ch, rch ) && IS_NPC( rch ) )
	{
	    strcat( buf, show_char_to_char_0( rch, ch ));
		
		number--;
		
		if( number == 0 )
		strcat( buf, "." );
		else if( number == 1 )
		strcat( buf, " and " );
		else
		strcat( buf, ", " );
	}
    }
	
	//printwrap( ch, buf, 72, NULL );
	
	
	strcat( buf, "\r\n" );
	
	if( !IS_NPC(ch) && IS_SET(ch->act, PLR_WRAP ) )
	printf_to_char( ch, "%s", buf );
	else
	{
	send_to_char( buf, ch );
	}
	
    return;
}



bool check_blind( CHAR_DATA *ch )
{
    if ( !IS_NPC(ch) && IS_SET(ch->act, PLR_HOLYLIGHT) )
	return TRUE;

    if ( IS_AFFECTED(ch, AFF_BLIND) )
    {
	send_to_char( "You can't see a thing!\r\n", ch );
	return FALSE;
    }

    return TRUE;
}

void do_look( CHAR_DATA *ch, char *argument )
{
    char arg1 [MAX_INPUT_LENGTH];
    char arg2 [MAX_INPUT_LENGTH];
    CHAR_DATA *victim;
    OBJ_DATA *obj;
    char *pdesc;

    if ( ch->desc == NULL )
	return;
	
    if ( !check_blind( ch ) )
	{
	send_to_char( "You are blind and cannot see a thing!\r\n", ch );
	return;
	}
	
    argument = one_argument( argument, arg1 );
    argument = one_argument( argument, arg2 );

    if ( arg1[0] == '\0' || !str_cmp( arg1, "auto" ) )
    {
		
	/* 'look' or 'look auto' */
	if( !IS_NPC(ch) && IS_SET( ch->act, PLR_COLOR ) )
	send_to_char( ROOMTITLE, ch );
	
	send_to_char( "[", ch );
	
	if( strlen(ch->in_room->area->name) > 2 && strlen(ch->in_room->name) > 2 )
	{
	send_to_char( ch->in_room->area->name, ch );
	send_to_char( ", ", ch );
	send_to_char( ch->in_room->name, ch );
	}
	else if ( strlen(ch->in_room->name) > 2 )
	send_to_char( ch->in_room->name, ch );
	else if ( strlen(ch->in_room->area->name) > 2 )
	send_to_char( ch->in_room->area->name, ch );
	
	if( IS_IMMORTAL( ch ) )
	printf_to_char( ch, " - %d]\r\n", ch->in_room->vnum );
	else
	send_to_char( "]\r\n", ch );
	
	if( !IS_NPC(ch) && IS_SET( ch->act, PLR_COLOR ) )
	send_to_char( NORMAL, ch );
	
	show_combined_to_char( ch->in_room->people, ch->in_room->contents, ch );
	show_char_to_char( ch->in_room->people,   ch );
	do_exits( ch, "" );
	return;
    }
	
    if ( !str_cmp( arg1, "i" ) || !str_cmp( arg1, "in" ) )
    {
	/* 'look in' */
	if ( arg2[0] == '\0' )
	{
	    send_to_char( "Look in what?\r\n", ch );
	    return;
	}

	if ( ( obj = get_obj_here( ch, arg2 ) ) == NULL )
	{
	    printf_to_char( ch, "I don't see any %s here.\r\n", arg2 );
	    return;
	}

	switch ( obj->item_type )
	{
	default:
	    printf_to_char( ch, "The %s does not appear to be a container.\r\n", smash_article(obj->short_descr) );
	    break;

	case ITEM_DRINK_CON:
	    if ( obj->value[1] <= 0 )
	    printf_to_char( ch, "The %s is empty.\r\n", smash_article(obj->short_descr) );
		else
	    printf_to_char( ch, "There is some liquid in the %s.\r\n", smash_article(obj->short_descr) );

	    break;

	case ITEM_CONTAINER:
	case ITEM_CORPSE_NPC:
	case ITEM_CORPSE_PC:
	
	    if ( IS_SET(obj->value[1], CONT_CLOSED) )
	    {
	    printf_to_char( ch, "The %s is closed.\r\n", smash_article(obj->short_descr) );
		break;
	    }

		new_show_list_to_char( obj->contains, ch, TRUE, TRUE, "It contains " );
		
	    break;
	}
	return;
    }
	
	if ( !str_cmp( arg1, "o" ) || !str_cmp( arg1, "on" ) )
    {
	/* 'look in' */
	if ( arg2[0] == '\0' )
	{
	    send_to_char( "Look on what?\r\n", ch );
	    return;
	}

	if ( ( obj = get_obj_here( ch, arg2 ) ) == NULL )
	{
	    printf_to_char( ch, "I don't see any %s here.\r\n", arg2 );
	    return;
	}

	switch ( obj->item_type )
	{
	default:
	    printf_to_char( ch, "There is nothing on %s.\r\n", smash_article(obj->short_descr) );
	    break;

	case ITEM_FURNITURE: case ITEM_MERCHANT_TABLE:
	
	    if ( IS_SET(obj->value[1], CONT_CLOSED) )
	    {
	    printf_to_char( ch, "The %s is closed.\r\n", smash_article(obj->short_descr) );
		break;
	    }

		new_show_list_to_char( obj->contains, ch, TRUE, TRUE, "On it, you see " );
		
	    break;
	}
	return;
    }

    if ( ( victim = get_char_room( ch, arg1 ) ) != NULL )
    {
		
		if( victim == ch )
		printf_to_char( ch, "At yourself?\r\n" );
	else		
		show_char_to_char_1( victim, ch );
	return;
    }

    for ( obj = ch->carrying; obj != NULL; obj = obj->next_content )
    {
	if ( can_see_obj( ch, obj ) )
	{
	    pdesc = get_extra_descr( arg1, obj->extra_descr );
	    if ( pdesc != NULL )
	    {
			
		if (!IS_NPC(ch) && IS_SET(ch->act, PLR_WRAP))
		printf_to_char( ch, "%s", pdesc );
		else
		send_to_char( pdesc, ch );
		return;
	    }

	    pdesc = get_extra_descr( arg1, obj->pIndexData->extra_descr );
	    if ( pdesc != NULL )
	    {
		if (!IS_NPC(ch) && IS_SET(ch->act, PLR_WRAP))
		printf_to_char( ch, "%s", pdesc );
		else
		send_to_char( pdesc, ch );
		return;
	    }
	}
    }

    for ( obj = ch->in_room->contents; obj != NULL; obj = obj->next_content )
    {
	if ( can_see_obj( ch, obj ) )
	{
	    pdesc = get_extra_descr( arg1, obj->extra_descr );
	    if ( pdesc != NULL )
	    {
		if (!IS_NPC(ch) && IS_SET(ch->act, PLR_WRAP))
		printf_to_char( ch, "%s", pdesc );
		else
		send_to_char( pdesc, ch );
		return;
	    }

	    pdesc = get_extra_descr( arg1, obj->pIndexData->extra_descr );
	    if ( pdesc != NULL )
	    {
		if (!IS_NPC(ch) && IS_SET(ch->act, PLR_WRAP))
		printf_to_char( ch, "%s", pdesc );
		else
		send_to_char( pdesc, ch );
		return;
	    }
	}
    }

    pdesc = get_extra_descr( arg1, ch->in_room->extra_descr );
    if ( pdesc != NULL )
    {
	if (!IS_NPC(ch) && IS_SET(ch->act, PLR_WRAP))
	printf_to_char( ch, "%s", pdesc );
	else
	send_to_char( pdesc, ch );
	return;
    }


    obj = get_obj_list( ch, arg1, ch->in_room->contents );
	
    if ( obj != NULL )
	{
	if( strlen(obj->description) > 2 )
	{
		if (!IS_NPC(ch) && IS_SET(ch->act, PLR_WRAP))
		{
		printf_to_char( ch, "%s", obj->description );
		send_to_char( "\r\n", ch );
		}
		else
		{
		send_to_char( obj->description, ch );
		send_to_char( "\r\n", ch );
		}
	return;
	}
	else
	{
	printf_to_char( ch, "You notice nothing unusual about %s.\r\n", obj->short_descr);
	return;
	}	
	}

	
	obj = get_obj_wear( ch, arg1 );
		
	if ( obj != NULL )
	{
	printf_to_char( ch, "You notice nothing unusual about your %s.\r\n", smash_article(obj->short_descr));
	return;
	}
	
	obj = get_obj_carry( ch, arg1 );
	
	if ( obj != NULL )
	{
	printf_to_char( ch, "You notice nothing unusual about your %s.\r\n", smash_article(obj->short_descr));
	return;
	}
	
	printf_to_char(ch, "I do not see %s here.\r\n", arg1 );
	
    return;
}





void do_examine( CHAR_DATA *ch, char *argument )
{
    char buf[MAX_STRING_LENGTH];
    OBJ_DATA *obj;
	//fixme add a bool to make sure we have output for all situations??

    if ( argument[0] == '\0' )
    {
	send_to_char( "Examine what?\r\n", ch );
	return;
    }
	
	obj = get_obj_here( ch, argument );
	
    if ( obj != NULL )
    {
	//fixme if feeling...then perception to detect specific verb traps???
	if ( obj->verb_trap != NULL )
	{
	send_to_char( "You get a distinct feeling that not all is as it seems.\r\n", ch );
	}	
		
	switch ( obj->item_type )
	{
	case ITEM_FURNITURE:
	//send_to_char( "When you look inside, you see:\r\n", ch );
	    sprintf( buf, "on %s", argument );
	    do_look( ch, buf );
		return;
	
	case ITEM_DRINK_CON:
	case ITEM_CONTAINER:
	case ITEM_CORPSE_NPC:
	case ITEM_CORPSE_PC:
	    //send_to_char( "When you look inside, you see:\r\n", ch );
	    sprintf( buf, "in %s", argument );
	    do_look( ch, buf );
		return;
	
	//fixme bool
	case ITEM_WEAPON: 
	if( obj->value[0] > -1 && obj->value[0] < MAX_WEAPON )
	printf_to_char( ch, "It is some kind of %s.\r\n", weapon_table[obj->value[0]].name ); 
	if( obj->st && obj->du )
	printf_to_char( ch, "You determine it has %d Strength and %d Durability.\r\n", obj->st, obj->du ); 
	return;
	
	case ITEM_ARMOR: 
	if( obj->value[3] > -1 && obj->value[3] < MAX_ARMOR )
	printf_to_char( ch, "It is some kind of %s.\r\n", armor_table[obj->value[3]].name ); //fixme customs??
	if( obj->st && obj->du )
	printf_to_char( ch, "You determine it has %d Strength and %d Durability.\r\n", obj->st, obj->du ); 
	return;
	
	case ITEM_SHIELD:
	if( obj->value[0] > -1 && obj->value[0] < MAX_SHIELD )
	printf_to_char( ch, "It is some kind of %s.\r\n", shield_table[obj->value[0]].name );
	if( obj->st && obj->du )
	printf_to_char( ch, "You determine it has %d Strength and %d Durability.\r\n", obj->st, obj->du ); 
	return;
	
	default:
	do_look( ch, obj->name );
	return;
	}
    }
	
	//fixme bool
	
	printf_to_char( ch, "Examine %s?\r\n", argument);

    return;
}




/*
 * Thanks to Zrin for auto-exit part.
 */
void do_exits( CHAR_DATA *ch, char *argument )
{
    extern char * const dir_name[];
    char buf[MAX_STRING_LENGTH];
    EXIT_DATA *pexit;
    bool found;
    int door;
	int exits = 0;

    buf[0] = '\0';

	/*
    if ( !check_blind( ch ) )
	return;
	*/
	
    strcpy( buf, IS_OUTSIDE(ch) ? "Obvious paths: " : "Obvious exits: " );

	for ( door = 0; door <= 10; door++ )
    {
	if ( ( pexit = ch->in_room->exit[door] ) != NULL
	&&   pexit->to_room != NULL )
	{
		exits++;
	}
    }	
	
    found = FALSE;
    for ( door = 0; door <= 10; door++ )
    {
	if ( ( pexit = ch->in_room->exit[door] ) != NULL
	&&   pexit->to_room != NULL )
	{
	    found = TRUE;

		strcat( buf, dir_name[door] );
		
		exits--;
		
		if( exits > 0 )
		strcat( buf, ", " );
	}
    }

    if ( !found )
	strcat( buf, "none" );

    send_to_char( buf, ch );
	send_to_char( "\r\n", ch );
    return;
}


void do_experience( CHAR_DATA *ch, char *argument )
{	

		printf_to_char(ch, "Level: %d  Deeds: %d%s\r\n", ch->level, ch->pcdata->deeds, ch->pcdata->deeds == 0 ? " (!)" : "" );
		printf_to_char(ch, "Experience: %d\r\n", ch->exp );
		printf_to_char(ch, "Exp. until next: %d\r\n",  get_total_exp_required( ch->level+1 ) - ch->exp );

		printf_to_char(ch, "Training points: %d PTP / %d MTP\r\n", !IS_NPC(ch) ? ch->pcdata->PTPs : 0,
			!IS_NPC(ch) ? ch->pcdata->MTPs : 0				);
					
		if( !IS_NPC(ch) )
		{
		float mind = (float) ch->pcdata->fieldExp / (float) get_max_field_exp(ch);
			
		if( mind >= 1.00 )
			printf_to_char( ch, "Your mind is completely saturated.  " );
		else if( mind > 0.90 )
			printf_to_char( ch, "Your mind is numbed.  " );
		else if( mind > 0.75 )
			printf_to_char( ch, "Your mind is becoming numbed.  " );
		else if( mind > 0.50 )
			printf_to_char( ch, "Your mind is muddled.  " );
		else if( mind > 0.25 )
			printf_to_char( ch, "Your mind is clear.  " );
		else if( mind > 0.00 )
			printf_to_char( ch, "Your mind is fresh and clear. " );
		else
			printf_to_char( ch, "Your mind is clear as a bell.  " );
		
		printf_to_char( ch, "\r\n" );

		//printf_to_char( ch, "Field exp.: %d/%d  ", ch->pcdata->fieldExp, get_max_field_exp(ch) );
		
		
		//if( EXP_BOOST != 1 )
		//printf_to_char( ch, "(%dx)\r\n", EXP_BOOST );
		}
		
}


void do_wealth( CHAR_DATA *ch, char *argument )
{
    printf_to_char( ch,
	"You rummage around in your pockets and determine that you have %d silver coins.\r\n",
	ch->gold );
	
	if( ch->gold > 1)
	act( "$n rummages around in $s pockets.", ch, NULL, NULL, TO_ROOM );	
	else
	act( "$n rummages around in $s pockets but comes up empty.", ch, NULL, NULL, TO_ROOM );	
}

void do_score2( CHAR_DATA *ch, char *argument )
{
    printf_to_char( ch, "Who is keeping score, and of what?\r\n" );

}

void do_credits( CHAR_DATA *ch, char *argument )
{
	do_help(ch, "diku");
}


void do_score( CHAR_DATA *ch, char *argument )
{
    char buf[MAX_STRING_LENGTH];

    sprintf( buf,
	"You are %s the %d-year-old %s %s.\r\nYou are level %d with %d experience.\r\n",
	IS_NPC(ch) ? ch->short_descr : ch->name,
	!IS_NPC(ch) ? ch->pcdata->age : 17,
	ch->sex == 1 ? "Male" : ch->sex == 2 ? "Female" : "Neuter",
	race_table[ch->race].name,
	ch->level,
	ch->exp );
    send_to_char( buf, ch );

    sprintf( buf,
   "    Strength (STR): %d (%d)\r\n   Endurance (END): %d (%d)\r\n   Dexterity (DEX): %d (%d)\r\n       Speed (SPD): %d (%d)\r\n   Willpower (WIL): %d (%d)\r\n     Potency (POT): %d (%d)\r\n    Judgment (JDG): %d (%d)\r\nIntelligence (INT): %d (%d)\r\n      Wisdom (WIS): %d (%d)\r\n       Charm (CHA): %d (%d)\r\n",
	get_curr_Strength(ch),
	stat_bonus(get_curr_Strength(ch), ch, STAT_STRENGTH),
	get_curr_Endurance(ch),
	stat_bonus(get_curr_Endurance(ch), ch, STAT_ENDURANCE),
	get_curr_Dexterity(ch),
	stat_bonus(get_curr_Dexterity(ch), ch, STAT_DEXTERITY),
	get_curr_Speed(ch),
	stat_bonus(get_curr_Speed(ch), ch, STAT_SPEED),
	get_curr_Willpower(ch),
	stat_bonus(get_curr_Willpower(ch), ch, STAT_WILLPOWER ),
	get_curr_Potency(ch),
	stat_bonus(get_curr_Potency(ch), ch, STAT_POTENCY ),
	get_curr_Judgement(ch),
	stat_bonus(get_curr_Judgement(ch), ch, STAT_JUDGEMENT ),
	get_curr_Intelligence(ch),
	stat_bonus(get_curr_Intelligence(ch), ch, STAT_INTELLIGENCE ),
	get_curr_Wisdom(ch),
	stat_bonus(get_curr_Wisdom(ch), ch, STAT_WISDOM ),
	get_curr_Charm(ch),
	stat_bonus(get_curr_Charm(ch), ch, STAT_CHARM ) );
    send_to_char( buf, ch );
	
	printf_to_char( ch,
	"Mana: %d/%d   Silver: %d\r\n",
	ch->mana, ch->max_mana, ch->gold );

    return;
}

char *spell_type( short target )
{
	switch (target)
	{
		case TAR_IGNORE: default: return "untargeted";
		case TAR_CHAR_OFFENSIVE: return "attack";
		case TAR_CHAR_DEFENSIVE: return "defense";
		case TAR_CHAR_HEALING: return "healing";
		case TAR_CHAR_SELF: return "self-cast";
		case TAR_OBJ_INV: return "utility";
		case TAR_EITHER: return "mixed";
	}
}


void do_spells( CHAR_DATA *ch, char *argument )
{

	if( argument[0] == '\0' )
	{
	printf_to_char(ch, "Type SPELLS ACTIVE to show your active spells " );
	printf_to_char(ch, "or SPELLS KNOWN to show your known spells.\r\n" );
	return;
	}
	
	if( !str_cmp( argument, "active" ) )
	{
	AFFECT_DATA *paf;
	
	if ( ch->affected != NULL )
    {
	send_to_char( "You currently have the following active spells:\r\n", ch );
	for ( paf = ch->affected; paf != NULL; paf = paf->next )
	{
	    printf_to_char( ch, "%s\r\n", spell_table[paf->type].name );
	}
    }

	return;
	}
	
	if( IS_NPC(ch) )
		return;

	if( !str_cmp( argument, "known" ) )
	{
	if( ch->pcdata->learned[gsn_spell_research] > 0 && ( ch->pcdata->learned[gsn_spirit_mana_control] > 0 || ch->pcdata->learned[gsn_elemental_mana_control] > 0 ) )
	send_to_char( "You can cast the following spells:\r\n", ch );

	{
	int i = 0;

	for( i = 0; i < MAX_SPELL; i++ )
	{
		if( can_cast(ch, i) == TRUE || IS_IMMORTAL(ch) )
		{
		printf_to_char( ch, "#%-5d Lv. %-5d %s %-24s (%s)\r\n", spell_table[i].number, spell_table[i].level, 
		spell_table[i].prof == CLASS_MAGE || spell_table[i].prof == CLASS_SORCERER ? "BLK" : "WHT",
		spell_table[i].name, spell_type(spell_table[i].target)  );
		}
	
	}
	}
	}
}

char *	const	day_name	[] =
{
	"Trepidation",
    "Sorrow", 
	"Caprice", 
	"Burdens", 
	"Wine", 
	"Feasts",
    "Triumph"
};

char *	const	month_name	[] =
{
    "Hope", 
	"Returning Sun", 
	"Divine Glory", 
	"Radiant Dawn",
	"Three Stars", 
	"Journeys", 
	"Reckoning", 
	"Last Fire",
	"Holy Hearth", 
	"Hunters", 
	"Blood", 
	"Death"
};

char *	const	daytime_name	[] =
{
	"midnight",
	"late at night",
	"late at night",
	"the witching hour",
	"early morning",
	"early morning",
	"early morning",
	"mid-morning",
	"mid-morning",
	"mid-morning",
	"late morning",
	"late morning",
	"noontime",
	"early afternoon",
	"afternoon",
	"afternoon",
	"late afternoon",
	"early evening",
	"early evening",
	"twilight",
	"evening",
	"evening",
	"late evening",
	"late evening"
};

void do_time( CHAR_DATA *ch, char *argument )
{
  time_t rawtime;
  struct tm * timeinfo;
  char *suf;
  int day;
    extern char str_boot_time[];


  time ( &rawtime );
  timeinfo = localtime ( &rawtime );
  
     day     = timeinfo->tm_mday;

         if ( day > 4 && day <  20 ) suf = "th";
    else if ( day % 10 ==  1       ) suf = "st";
    else if ( day % 10 ==  2       ) suf = "nd";
    else if ( day % 10 ==  3       ) suf = "rd";
    else                             suf = "th";
  
  //printf_to_char ( ch, "System time: %s\r\n\r\n", asctime (timeinfo) );
  
  printf_to_char( ch, "Today is the Day of %s, the %d%s of the month of %s, %d years after Tholamir's Deposing.\r\n", day_name[timeinfo->tm_wday],
  timeinfo->tm_mday, suf, month_name[timeinfo->tm_mon], timeinfo->tm_year );
  
  printf_to_char( ch, "It is %d:%s%d, making it %s.\r\n", timeinfo->tm_hour, timeinfo->tm_min < 9 ? "0" : "", timeinfo->tm_min,
	daytime_name[timeinfo->tm_hour]);
	
	if( IS_IMMORTAL(ch) )
	{
	printf_to_char( ch, "Boot time (local):    %s\rCurrent time (local): %s\r",
	str_boot_time,
	(char *) ctime( &current_time ));
	}

  return;
}

char *temp_string( int temp )
{
	if( temp > 90 )
		return "blazing hot";
	else if( temp > 80 )
		return "hot and muggy";
	else if( temp > 75 )
		return "nice and balmy";
	else if( temp > 70 )
		return "pleasantly warm";
	else if( temp > 65 )
		return "very comfortable";
	else if( temp > 60 )
		return "cool";
	else if( temp > 50 )
		return "a bit nippy";
	else if( temp > 40 )
		return "pretty cold";
	else if( temp > 32 )
		return "cold";
	else if( temp > 20 )
		return "freezing cold";
	else
		return "bone-chillingly frigid";
}

void do_weather( CHAR_DATA *ch, char *argument )
{
    if ( !IS_OUTSIDE(ch) )
    {
	send_to_char( "You are indoors so it is hard to tell what the weather outside is.\r\n", ch );
	return;
    }
	
	// something about sunlight, the moon, etc...  fixme

	printf_to_char( ch, "It is %s.\r\n", temp_string( weather_info.temp ) );
	
	if( weather_info.sky == SKY_RAINING )
	{
		if( weather_info.temp > 32 )
		printf_to_char( ch, "It is raining.\r\n" );
		else
		printf_to_char( ch, "It is snowing.\r\n" );
	}
	
	if( IS_IMMORTAL( ch ) )
	{
		printf_to_char( ch, "Temperature   : %d F.\r\n", weather_info.temp );
		printf_to_char( ch, "Today's high  : %d F.\r\n", weather_info.todays_high );
		printf_to_char( ch, "Tonight's low : %d F.\r\n", weather_info.tonights_low );
		printf_to_char( ch, "Precip. start : %d:%s%d.\r\n", weather_info.precip_start_hour, weather_info.precip_start_minute < 9 ? "0" : "", weather_info.precip_start_minute );
		printf_to_char( ch, "Precip. end   : %d:%s%d.\r\n", weather_info.precip_end_hour, weather_info.precip_end_minute < 9 ? "0" : "", weather_info.precip_end_minute );
	}
	
    return;
}


void do_makeitrain( CHAR_DATA *ch, char *argument )
{
  time_t rawtime;
  struct tm * timeinfo;

  time ( &rawtime );
  timeinfo = localtime ( &rawtime );

  weather_info.precip_start_hour = timeinfo->tm_hour,
  weather_info.precip_start_minute = timeinfo->tm_min+1;
  
  return;
}



void do_help( CHAR_DATA *ch, char *argument )
{
    char argall[MAX_INPUT_LENGTH];
    char argone[MAX_INPUT_LENGTH];
	bool nothing=TRUE;
    HELP_DATA *pHelp;
	int i;

    if ( argument[0] == '\0' )
		argument = "summary";

	if ( strlen(argument) < 2 )
	{
		send_to_char("If you want help, you need to be more specific.\r\n", ch );
		return;
	}

    /*
     * Tricky argument handling so 'help a b' doesn't match a.
     */
    argall[0] = '\0';
    while ( argument[0] != '\0' )
    {
	argument = one_argument( argument, argone );
	if ( argall[0] != '\0' )
	    strcat( argall, " " );
	strcat( argall, argone );
    }

    for ( pHelp = help_first; pHelp != NULL; pHelp = pHelp->next )
    {
	 if ( pHelp->level > get_trust( ch ) )
	    continue;

	if ( is_name( argall, pHelp->keyword ) )
	{
	    if ( pHelp->level >= 0 ) // && str_cmp( argall, "imotd" ) )
	    {
		printf_to_char( ch, "<<< HELP: %s >>>\r\n", pHelp->keyword );
		nothing=FALSE;
	    }
	    if ( pHelp->text[0] == '.' )
		{
			
	    send_to_char( pHelp->text+1, ch );

		nothing=FALSE;
	    } 
		else
		{
	    send_to_char( pHelp->text, ch );
	
		nothing=FALSE;
		}
	}
    } 
	
	for( i = 0; cmd_table[i].name[0] != '\0'; i++ )
	{
		if( is_name( argall, cmd_table[i].name ) && cmd_table[i].level <= ch->level )
		{
		nothing=FALSE;
		printf_to_char( ch, "\r\n---\r\nCommand \"%s\" =FLAGS: ", cmd_table[i].name );
		
		if( IS_SET(cmd_table[i].flags, CMD_DONT_PARSE) )
			printf_to_char( ch, "(unparsed) " );
		
		if( IS_SET(cmd_table[i].flags, COMMAND_OLC) )
			printf_to_char( ch, "#Online Creation# " );
		
		if( IS_SET(cmd_table[i].flags, BYPASS_STUN) )
			printf_to_char( ch, "[bypasses stun] " );
		
		if( IS_SET(cmd_table[i].flags, BYPASS_RT) )
			printf_to_char( ch, "[bypasses RT] " );
		
		if( IS_SET(cmd_table[i].flags, CMD_BREAKS_PREDELAY) )
			printf_to_char( ch, "(breaks concentration) " );
		
		if( IS_SET(cmd_table[i].flags, CMD_BREAKS_HIDE) )
			printf_to_char( ch, "(stops hiding) " );
		
		if( IS_SET(cmd_table[i].log, LOG_NEVER) )
			printf_to_char( ch, "*never logged* " );

		if( IS_SET(cmd_table[i].log, LOG_ALWAYS) )
			printf_to_char( ch, "*always logged* " );
		
		printf_to_char( ch, "\r\n---\r\n" );
		}
	}
	
	for( i = 0; i < MAX_SPELL; i++ )
	{
		if( is_name( argall, spell_table[i].name ) )
		{		
		printf_to_char( ch, "\r\n===\r\n%s (%d)\r\n", spell_table[i].name, spell_table[i].number );
		printf_to_char( ch, "Level %d %s  ", spell_table[i].level , spell_table[i].prof == CLASS_MAGE || spell_table[i].prof == CLASS_SORCERER ? "Black" : "White");

		switch ( spell_table[i].target )
		{
			case TAR_IGNORE: printf_to_char( ch, "General Spell\r\n" ); break;
			case TAR_CHAR_OFFENSIVE: printf_to_char( ch, "Offensive Spell\r\n" ); break;
			case TAR_CHAR_DEFENSIVE: printf_to_char( ch, "Defensive Spell\r\n" ); break;
			case TAR_CHAR_SELF: printf_to_char( ch, "Self-cast Spell\r\n" ); break;
			case TAR_OBJ_INV: printf_to_char( ch, "Object Spell\r\n" ); break;
			default: break;
		}
		
		printf_to_char( ch, "\r\n===\r\n" );
		
		nothing = FALSE;
		}
		
	}
	
	for( i = 0; i < MAX_SKILL; i++ )
	{
		if( is_name( argall, skill_table[i].name ) )
		{		
		printf_to_char( ch, "\r\n***\r\n%s [skill]\r\n", skill_table[i].name );
		printf_to_char( ch, "Base Training Cost: %d Phys. TPs/%d Ment. TPs\r\n\r\n", skill_table[i].basePTPs, skill_table[i].baseMTPs );

		switch ( skill_table[i].type )
		{
			case SKILL_GENERAL: send_to_char( "General skill ", ch ); break;
			case SKILL_COMBAT: send_to_char( "Combat skill ", ch );  break;
			case SKILL_MAGIC: send_to_char( "Magic skill ", ch ); break;
			case SKILL_STEALTH: send_to_char( "Stealth skill ", ch );  break;
			case SKILL_NOT_IMPLEMENTED: send_to_char( "Not yet implemented skill ", ch );  break;
			default: send_to_char( "Bugged skill ", ch ); break;
		}
		
		switch( skill_table[i].difficulty )
		{
		case DIFFICULTY_TRIVIAL:
		case DIFFICULTY_EASY:
			printf_to_char( ch, "that can be trained thrice per level.\r\n" );
			break;
	
		case DIFFICULTY_HARD:
		case DIFFICULTY_ARDUOUS:
			printf_to_char( ch, "that can be trained once per level.\r\n" );
			break;
	
		default:
		case DIFFICULTY_NORMAL:
		
			printf_to_char( ch, "that can be trained twice per level.\r\n" );
		}
		nothing = FALSE;
		printf_to_char( ch, "\r\n***\r\n" );
		}
		
	}
	
	//stats
	for( i = 0; i < MAX_RACE; i++ )
	{
		if( is_name( argall, race_table[i].name ) )
		{		
		printf_to_char( ch, "\r\n____\r\nRace \"%s\"\r\n", race_table[i].name );
		printf_to_char( ch, "Average life expectancy : %d years\r\n", race_table[i].lifespan );
		
		stat_bonus( 50, ch, STAT_STRENGTH );
		
		printf_to_char( ch, "Max Health : %d\r\n", race_table[i].max_Health );
		printf_to_char( ch, "Encumbrance Factor : %d\r\n", race_table[i].encumbrance_Factor );
		printf_to_char( ch, "Max Health : %d\r\n", race_table[i].max_Health );
		printf_to_char( ch, "\r\n____\r\n" );
		nothing = FALSE;
		}
		
		
	}
	
	/*
	struct	social_type
{
    char * const	name;
    char * const	char_no_arg;
    char * const	others_no_arg;
    char * const	char_found;
    char * const	others_found;
    char * const	vict_found;
    char * const	char_auto;
    char * const	others_auto;
	char * const	obj_inv_self; // socials that affect objects just like in GS!
    char * const	obj_inv_room;
	char * const	obj_room_self;
    char * const	obj_room_room;
};
*/
	
	for( i = 0; social_table[i].name[0] != '\0'; i++ )
	{
		if( is_name( argall, social_table[i].name ) )
		{
		nothing=FALSE;
		printf_to_char( ch, "\r\n+++\r\nFluff Command \"%s\" ... ", social_table[i].name );
		
		if( social_table[i].char_no_arg != NULL  )
			printf_to_char( ch, "No arguments: %s ", social_table[i].char_no_arg  );
		
		printf_to_char( ch, "\r\n+++\r\n" );
		}
	}
	
	//weapons, armors, gems??
	
	if( nothing )
	{
    send_to_char( "No help on that word.\r\nSyntax:  HELP (command), (skill), (spell), (race), etc.\r\n", ch );

	{
	int col;
	char buf[MSL];
	
	send_to_char("Other help files:\n\r", ch );

	col = 0;
	for ( pHelp = help_first ; pHelp != NULL ; pHelp = pHelp->next )
	{
	    if ( pHelp->level != -1 && pHelp->level <= get_trust(ch) ) // -1 never displayed
	    {
		sprintf( buf, "%-25s", pHelp->keyword );
		send_to_char( buf, ch );
		if (++col % 3 == 0)
		    send_to_char("\n\r", ch );
	    }
	}

	send_to_char("\n\r", ch );
	}
	}

    return;
}


//who vs who full
void do_who( CHAR_DATA *ch, char *argument )
{
    char buf[MAX_STRING_LENGTH];
    DESCRIPTOR_DATA *d;
	int nMatch = 0;
	
	strcpy( buf, "" );
	
    for ( d = descriptor_list; d != NULL; d = d->next )
    {
	CHAR_DATA *wch;
	
	if ( d->connected != CON_PLAYING || !can_see( ch, d->character ) )
	    continue;

	wch   = ( d->original != NULL ) ? d->original : d->character;

	nMatch++;

	/*
	 * Format it up.
	 */
	if( !str_cmp(argument, "full"))
	{
	strcat( buf, wch->name );
	strcat( buf, "\r\n" );
    }
	}

	if( nMatch > 0 )
	{
    send_to_char( buf, ch );
	send_to_char( "\r\n", ch );
	}
	
	printf_to_char( ch, "%d player%s.\r\n", nMatch, nMatch == 1 ? "" : "s" );
	
	if( str_cmp(argument, "full"))
		printf_to_char( ch, "WHO FULL will show character names.\r\n" );
	
    return;
}



void do_inventory( CHAR_DATA *ch, char *argument )
{
    //send_to_char( "You are carrying:\r\n", ch );
    //show_list_to_char( ch->carrying, ch, TRUE, TRUE );
	new_show_list_to_char( ch->carrying, ch, TRUE, FALSE, "You are carrying " );
    return;
}


void do_equipment( CHAR_DATA *ch, char *argument )
{
	show_eq_to_char( ch, ch, TRUE, TRUE );
	do_inventory( ch, "" );
}

void do_compare( CHAR_DATA *ch, char *argument )
{
    return;
}


void do_where( CHAR_DATA *ch, char *argument )
{
    return;
}



//fixme appraise
void do_consider( CHAR_DATA *ch, char *argument )
{
	return;
}








void do_password( CHAR_DATA *ch, char *argument )
{
    char arg1[MAX_INPUT_LENGTH];
    char arg2[MAX_INPUT_LENGTH];
    char *pArg;
    char *pwdnew;
    char *p;
    char cEnd;

    if ( IS_NPC(ch) )
	return;

    /*
     * Can't use one_argument here because it smashes case.
     * So we just steal all its code.  Bleagh.
     */
    pArg = arg1;
    while ( isspace((unsigned char) *argument) )
	argument++;

    cEnd = ' ';
    if ( *argument == '\'' || *argument == '"' )
	cEnd = *argument++;

    while ( *argument != '\0' )
    {
	if ( *argument == cEnd )
	{
	    argument++;
	    break;
	}
	*pArg++ = *argument++;
    }
    *pArg = '\0';

    pArg = arg2;
    while ( isspace((unsigned char) *argument) )
	argument++;

    cEnd = ' ';
    if ( *argument == '\'' || *argument == '"' )
	cEnd = *argument++;

    while ( *argument != '\0' )
    {
	if ( *argument == cEnd )
	{
	    argument++;
	    break;
	}
	*pArg++ = *argument++;
    }
    *pArg = '\0';

    if ( arg1[0] == '\0' || arg2[0] == '\0' )
    {
	send_to_char( "Syntax: password <old> <new>.\r\n", ch );
	return;
    }

    if ( strcmp( crypt( arg1, ch->pcdata->pwd ), ch->pcdata->pwd ) )
    {
	send_to_char( "Wrong password.\r\n", ch );
	add_roundtime( ch, 40 );
	show_roundtime( ch, 40 );
	return;
    }

    if ( strlen(arg2) < 5 )
    {
	send_to_char(
	    "New password must be at least five characters long.\r\n", ch );
	return;
    }

    /*
     * No tilde allowed because of player file format.
     */
    pwdnew = crypt( arg2, ch->name );
    for ( p = pwdnew; *p != '\0'; p++ )
    {
	if ( *p == '~' )
	{
	    send_to_char(
		"New password not acceptable, try again.\r\n", ch );
	    return;
	}
    }

    free_string( ch->pcdata->pwd );
    ch->pcdata->pwd = str_dup( pwdnew );
    save_char_obj( ch );
    send_to_char( "Ok.\r\n", ch );
    return;
}





/*
 * Contributed by Alander.
 */
 
//socials now work on objects in room and in inventory, combined COMMAND to come


void do_commands( CHAR_DATA *ch, char *argument )
{
    char buf[MAX_STRING_LENGTH];
    int cmd;
    int col;
 
    col = 0;
    for ( cmd = 0; cmd_table[cmd].name[0] != '\0'; cmd++ )
    {
        if ( cmd_table[cmd].level <  LEVEL_HERO
        &&   cmd_table[cmd].level <= get_trust( ch ) )
	{
	    sprintf( buf, "%-12s", cmd_table[cmd].name );
	    send_to_char( buf, ch );
	    if ( ++col % 6 == 0 )
		send_to_char( "\n\r", ch );
	}
    }
 
    if ( col % 6 != 0 )
	send_to_char( "\n\r", ch );
    return;
}

void do_socials( CHAR_DATA *ch, char *argument )
{
    char buf[MAX_STRING_LENGTH];
    int iSocial;
    int col;
 
    col = 0;
    for ( iSocial = 0; social_table[iSocial].name[0] != '\0'; iSocial++ )
    {
	sprintf( buf, "%-12s", social_table[iSocial].name );
	send_to_char( buf, ch );
	if ( ++col % 6 == 0 )
	    send_to_char( "\n\r", ch );
    }
 
    if ( col % 6 != 0 )
	send_to_char( "\n\r", ch );
    return;
}



void do_channels( CHAR_DATA *ch, char *argument )
{
   return;
}

void do_wrap( CHAR_DATA *ch, char *argument )
{
	  short value = is_number( argument ) ? atoi( argument ) : 77;
	  
	  if (IS_NPC(ch) )
		  return;
	  
	  if( !IS_SET(ch->act, PLR_WRAP) )
	  {
		do_config(ch, "wrap");
		return;
	  }
	  
	  if( value < 30 || value > 120 )
	  {
		  printf_to_char( ch, "Wrap value must be between 30 and 120.\r\n" );
		  return;
	  }
	  
	  ch->pcdata->wrap = value;
	  printf_to_char( ch, "Wrap set to %d characters.\r\n", value );
}

	

/*
 * Contributed by Grodyn.
 */
void do_config( CHAR_DATA *ch, char *argument )
{
    char arg[MAX_INPUT_LENGTH];

    if ( IS_NPC(ch) )
	return;

    one_argument( argument, arg );

    if ( arg[0] == '\0' )
    {
		printf_to_char( ch, "%10s %10s - GAME BEHAVIOR\r\n\r\n", "SETTING", "TOGGLED");
		
		if( IS_SET(ch->act, PLR_BLANK) )
		printf_to_char( ch, "%10s %10s - Blank line.\r\n", "Blank", "ON" );
		else
		printf_to_char( ch, "%10s %10s - No blank line.\r\n", "Blank", "OFF");	
	
		if( IS_SET(ch->act, PLR_BRIEF) )
		printf_to_char( ch, "%10s %10s - No room descs.\r\n", "Brief", "ON" );
		else
		printf_to_char( ch, "%10s %10s - Room descs.\r\n", "Brief", "OFF");	
		
		if( IS_SET(ch->act, PLR_COINS) )
		printf_to_char( ch, "%10s %10s - Picking up coins.\r\n", "Coins", "ON");
		else
		printf_to_char( ch, "%10s %10s - Pick up coins.\r\n", "coins", "OFF");	
	
		if( IS_SET(ch->act, PLR_COLOR) )
		printf_to_char( ch, "%10s %10s - Color is on.\r\n", "Color", "ON");
		else
		printf_to_char( ch, "%10s %10s - Color is off.\r\n", "Color", "OFF");	
	
		if( IS_SET(ch->act, PLR_ECHO) )
		printf_to_char( ch, "%10s %10s - Input is echoed.\r\n", "Echo", "ON" );
		else
		printf_to_char( ch, "%10s %10s - Input is not echoed.\r\n", "Echo", "OFF");	
	
		if( IS_SET(ch->act, PLR_PARSE) )
		printf_to_char( ch, "%10s %10s - Natural language parser.\r\n", "Parser", "ON" );
		else
		printf_to_char( ch, "%10s %10s - No parser.\r\n", "Parser", "OFF");	


		if( IS_SET(ch->act, PLR_PROMPT) )
		printf_to_char( ch, "%10s %10s - Prompt is shown.\r\n", "Prompt", "ON" );
		else
		printf_to_char( ch, "%10s %10s - No prompt.\r\n", "Prompt", "OFF");	
	
	
		if( IS_SET(ch->act, PLR_RETURN) )
		printf_to_char( ch, "%10s %10s - Return and newline.\r\n", "Return", "ON" );
		else
		printf_to_char( ch, "%10s %10s - No return and newline.\r\n", "Return", "OFF");	
	
		if( IS_SET(ch->act, PLR_TELNET_GA) )
		printf_to_char( ch, "%10s %10s - Telnet GA will be sent.\r\n", "TelnetGA", "ON" );
		else
		printf_to_char( ch, "%10s %10s - No Telnet GA will be sent.\r\n", "TelnetGA", "OFF");	
	
		if( IS_SET(ch->act, PLR_WRAP) )
		printf_to_char( ch, "%10s %10s - Some text wrapped (%d chars).\r\n", "Wrap", "ON", ch->pcdata->wrap);
		else
		printf_to_char( ch, "%10s %10s - All text unwrapped.\r\n", "Wrap", "OFF");	


		/*

	send_to_char(  IS_SET(ch->act, PLR_SILENCE)
	    ? "silence\tOON		You are silenced.\r\n"
	    : ""
	    , ch );

	send_to_char( !IS_SET(ch->act, PLR_NO_EMOTE)
	    ? ""
	    : "emote\tOOFF     You cannot emote.\r\n"
	    , ch );

	send_to_char( !IS_SET(ch->act, PLR_NO_TELL)
	    ? ""
	    : "tell\tOOFF     You cannot tell.\r\n"
	    , ch );
		
		*/
		
	printf_to_char(ch, "\r\nType SET <option> to toggle on/off.\r\n\r\n");
    }
    else
    {
	int bit;

	     if ( is_name( arg, "color"    ) ) bit = PLR_COLOR;
	else if ( is_name( arg, "coins"    ) ) bit = PLR_COINS;
	else if ( is_name( arg, "blank"    ) ) bit = PLR_BLANK;
	else if ( is_name( arg, "wrap"    ) ) bit = PLR_WRAP;
	else if ( is_name( arg, "brief"    ) ) bit = PLR_BRIEF;
	else if ( is_name( arg, "parser"    ) ) bit = PLR_PARSE;
	else if ( is_name( arg, "echo"    ) ) bit = PLR_ECHO;
	else if ( is_name( arg, "return"  ) ) bit = PLR_RETURN;
    else if ( is_name( arg, "prompt"   ) ) bit = PLR_PROMPT;
	else if ( is_name( arg, "telnetga" ) ) bit = PLR_TELNET_GA;
	else
	{
	    send_to_char( "Set which option on or off?\r\n", ch );
	    return;
	}

	if ( !IS_SET( ch->act, bit ) )
	{
	    SET_BIT    (ch->act, bit);
			send_to_char( "Setting turned on.\r\n", ch );
			
			if( bit == PLR_WRAP )
			{
				ch->pcdata->wrap = 77;
				printf_to_char(ch, "(Wrap value currently set to %d characters.)\r\n(Use WRAP <#> to set specific length.)\r\n", ch->pcdata->wrap);
			}
	}
	else
	{	
    REMOVE_BIT (ch->act, bit);
		send_to_char( "Setting turned off.\r\n", ch );
	}
	
    }
    return;
}


void do_health( CHAR_DATA *ch, char *argument )
{
	char buf[MAX_STRING_LENGTH];
	bool found = FALSE;
	int iBody = 1;
	
	strcpy( buf, "" );
	
	while( iBody < MAX_BODY )
	{
		if( ch->body[iBody] > 0 && ch->body[iBody] < 4 )
		{
			strcat( buf, "Your " );
			strcat( buf, body_name[iBody] );
			strcat( buf, " is " );
			
			if( ch->body[iBody] == 1 )
				strcat( buf, "lightly wounded.  ");
			else if( ch->body[iBody] == 2 )
				strcat( buf, "seriously wounded.  ");
			else if( ch->body[iBody] == 3 )
				strcat( buf, "critically wounded.  ");
			
			found = TRUE;
		}	
	iBody++;
	}
	
	iBody = 0;
	
	while( iBody < MAX_BODY )
	{
		if( ch->scars[iBody] > 0 && ch->scars[iBody] < 4 )
		{
			strcat( buf, "Your " );
			strcat( buf, body_name[iBody] );
			strcat( buf, " is " );
			
			if( ch->scars[iBody] == 1 )
				strcat( buf, "faintly scarred.  ");
			else if( ch->scars[iBody] == 2 )
				strcat( buf, "moderately scarred.  ");
			else if( ch->scars[iBody] == 3 )
				strcat( buf, "heavily wounded.  ");
			
			found = TRUE;
		}	
	iBody++;
	}
	
	if( found )
	{
	if( !IS_NPC(ch) && IS_SET( ch->act, PLR_WRAP ) )
	printf_to_char( ch, "%s", buf );
	else
	{
	send_to_char( buf, ch );
	}
	}
	else
	send_to_char("You seem to be in one piece.", ch );

    
	iBody = 0;
	
	
	//
	//Bleeding:
    //Area Health per Round
    //----------------------------------
    //Right leg 4
    //Chest 4
	//
	while( iBody < MAX_BODY )
	{
		if( ch->bleed[iBody] > 0 )
			printf_to_char( ch, "Your %s is bleeding at %d per.%s  ", body_name[iBody], ch->bleed[iBody], ch->bandage[iBody] > 0 ? " (Bandaged)" : "");
		
	iBody++;
	}
	
	
	printf_to_char(ch, "\r\n\r\n    Maximum Health Points: %d\r\n  Remaining Health Points: %d\r\r\n\n    Maximum Spirit Points: %d\r\n  Remaining Spirit Points: %d\r\n", ch->max_hit, ch->hit, ch->max_move, ch->move );
	
	return;
}

void do_mana( CHAR_DATA *ch, char *argument )
{
	printf_to_char(ch, "Mana Points: %d  Remaining: %d\r\n", ch->max_mana, ch->mana );
}

int base_carrying_capacity(CHAR_DATA *ch)
{
	float a,b,c,d;
	int e;
	
	a = (float) get_curr_Strength(ch) - 20.00;
	
	b = a / 200.00;
	
	c = b * (float) body_weight(ch);
	
	d = c + ((float) body_weight(ch) / 200.00);
	
	e = (int) d;
	
	return e;
}

int get_encumbrance_level( CHAR_DATA *ch )
{
	float e;
	float s;
	
	s = (float) ch->gold / 160.00;

	e = ((float) ch->carry_weight + s) - (float) base_carrying_capacity(ch);
	
	if (e < 1 )
		e = 0;
	else
	{
		e = e / (float) body_weight( ch ) * 100.00;
	}
	
	return (int) e;
}
	

void do_encumbrance(CHAR_DATA *ch, char *argument )
{
	float e;
	float s;
	
	if( IS_NPC(ch) )
		return;
	
	s = (float) ch->gold / 160.00;
	
	//printf_to_char(ch, "Base carrying capacity: %d\r\n", base_carrying_capacity(ch) );
	//printf_to_char(ch, "Your gear weighs: %d    plus silver weight: %f\r\n", ch->carry_weight,s );

	e = ((float) ch->carry_weight + s) - (float) base_carrying_capacity(ch);
	
	if (e < 1 )
		e = 0;
	else
	{
		e = e / (float) body_weight( ch ) * 100.00;
	}
	
	//printf_to_char(ch, "Your encumbrance level is: %f\r\n", e );
	
	if( e == 0 )
    printf_to_char(ch, "You are not encumbered enough to notice.\r\n");
else if( e < 11 )
    printf_to_char(ch, "Your load is a bit heavy.\r\n");
else if( e < 21 )
    printf_to_char(ch, "You feel somewhat weighed down, but can still move well.\r\n");
else if( e < 31 )
    printf_to_char(ch, "You are definitely feeling the effects of the weight you are carrying.\r\n");
else if( e < 41 )
    printf_to_char(ch, "Your shoulders sag under the weight of your gear, and your reactions are slowed.\r\n");
else if( e < 51 )
    printf_to_char(ch, "The weight you are carrying is giving you a backache.\r\n");
else if( e < 66 )
    printf_to_char(ch, "You are beginning to stoop under the load you are carrying, and your movement is slow.\r\n");
else if( e < 81 )
    printf_to_char(ch, "It is difficult to move quickly, and your legs ache from the load.\r\n");
else if( e < 101 )
    printf_to_char(ch, "You find it nearly impossible to move around, and you ache all over from the load you are hauling.\r\n");
else
    printf_to_char(ch, "You are so weighed down with stuff you can barely move.\r\n");

	return;
}


void do_age( CHAR_DATA *ch, char *argument )
{
	short input;
	
	if( IS_NPC(ch) )
		return;
	else
		input =  atoi(argument);
	
	if( ch->in_room->vnum != ROOM_VNUM_START  && !IS_IMMORTAL( ch ) )
	{
	    send_to_char( "It is too late to change your age.\r\n", ch );
	    return;
	}		
	
	if( input >= race_table[ch->race].lifespan / 4 && input < race_table[ch->race].lifespan )
	{
		ch->pcdata->age = input;
		
		printf_to_char( ch, "You now appear to be %d years old.\r\n", ch->pcdata->age );
		return;
	}
	else 
	{
		printf_to_char( ch, "You currently appear to be %d years old.\r\n", ch->pcdata->age );
		return;
	}
	
}

void do_hair( CHAR_DATA *ch, char *argument )
{
		if (IS_NPC(ch))
		return;
	
	if( ch->in_room->vnum != ROOM_VNUM_START && !IS_IMMORTAL( ch )  )
	{
	    send_to_char( "It is too late to change your hair color.\r\n", ch );
	    return;
	}		

    if ( argument[0] != '\0' )
    {
	smash_tilde( argument );
	
	if( strlen(argument) < 3 )
	{
	    send_to_char( "Please put SOME thought into your hair color.\r\n", ch );
	    return;
	}	
	
	if ( strlen(argument) > 20 )
	{
	    send_to_char( "Your hair color can't be THAT complicated.\r\n", ch );
	    return;
	}

	free_string( ch->pcdata->hair );
	ch->pcdata->hair = str_dup( argument );
    }

	printf_to_char( ch, "You now have %s hair.\r\n", ch->pcdata->hair );

    return;
}

void do_eyes( CHAR_DATA *ch, char *argument )
{
		if (IS_NPC(ch))
		return;
	
	if( ch->in_room->vnum != ROOM_VNUM_START && !IS_IMMORTAL( ch )  )
	{
	    send_to_char( "It is too late to change your eye color.\r\n", ch );
	    return;
	}		

    if ( argument[0] != '\0' )
    {
	smash_tilde( argument );
	
	if( strlen(argument) < 3 )
	{
	    send_to_char( "Please put SOME thought into your eye color.\r\n", ch );
	    return;
	}	
	
	if ( strlen(argument) > 20 )
	{
	    send_to_char( "Your eye color can't be THAT complicated.\r\n", ch );
	    return;
	}

	free_string( ch->pcdata->eyes );
	ch->pcdata->eyes = str_dup( argument );
    }

	printf_to_char( ch, "You now have %s eyes.\r\n", ch->pcdata->eyes );

    return;
}

void do_skincolor( CHAR_DATA *ch, char *argument )
{
	if (IS_NPC(ch))
		return;
	
	if( ch->in_room->vnum != ROOM_VNUM_START && !IS_IMMORTAL( ch ) )
	{
	    send_to_char( "It is too late to change your skin color.\r\n", ch );
	    return;
	}		

    if ( argument[0] != '\0' )
    {
	smash_tilde( argument );
	
	if( strlen(argument) < 3 )
	{
	    send_to_char( "Please put SOME thought into your skin color.\r\n", ch );
	    return;
	}	
	
	if ( strlen(argument) > 20 )
	{
	    send_to_char( "Your skin color can't be THAT complicated.\r\n", ch );
	    return;
	}

	free_string( ch->pcdata->skin );
	ch->pcdata->skin = str_dup( argument );
    }

	printf_to_char( ch, "You now have %s skin.\r\n", ch->pcdata->skin );

    return;
}

char *pers( CHAR_DATA *ch, CHAR_DATA *looker  )
{	
	if( !can_see( looker, ch ) )
		return "someone";
		
	if( !IS_NPC(ch) )
	{
		return ch->name;
	}
	else
	{
		static char buf[MSL];
		sprintf( buf, "{m%s{x", ch->short_descr ); // monsterbold!
		return buf;
	}

}


char *cap_pers( CHAR_DATA *ch, CHAR_DATA *looker  )
{	
	if( !can_see( looker, ch ) )
		return "someone";
		
	if( !IS_NPC(ch) )
	{
		return ch->name;
	}
	else
	{
		static char buf[MSL];
		sprintf( buf, "{m%s{x", capitalize(ch->short_descr) ); // monsterbold!
		return buf;
	}

}


#if defined(macintosh)
#include <types.h>
#else
#include <sys/types.h>
#endif
#include <stdio.h>
#include <string.h>
#include <time.h>
#include "merc.h"


char *	const	dir_name	[]		=
{
    "north", 		// 0
	"east", 		// 1
	"south", 		// 2
	"west", 		// 3
	"up", 			// 4
	"down", 		// 5
	"northeast",	// 6
	"southeast",	// 7
	"southwest",	// 8
	"northwest",	// 9
	"out"			// 10
};

const	short	rev_dir		[]		=
{
     2, 3, 0, 1, 5, 4, 
	 DIR_SOUTHWEST, DIR_NORTHWEST, DIR_NORTHEAST, DIR_SOUTHEAST, 10
};




/*
 * Local functions.
 */
int	find_door	args( ( CHAR_DATA *ch, char *arg ) );
bool	has_key		args( ( CHAR_DATA *ch, int key ) );



void move_char( CHAR_DATA *ch, int door )
{
    CHAR_DATA *fch;
    CHAR_DATA *fch_next;
    ROOM_INDEX_DATA *in_room;
    ROOM_INDEX_DATA *to_room;
    EXIT_DATA *pexit;

    if ( door < 0 || door > 10 )
    {
	bug( "Do_move: bad door %d.", door );
	return;
    }

    in_room = ch->in_room;
    if ( ( pexit   = in_room->exit[door] ) == NULL
    ||   ( to_room = pexit->to_room      ) == NULL )
    {
	send_to_char( "You can't go there.\r\n", ch );
	return;
    }

    if ( room_is_private( to_room ) )
    {
	send_to_char( "That room is private right now.\r\n", ch );
	return;
    }

    if ( !IS_NPC(ch) )
    {
	if ( in_room->sector_type == SECT_AIR
	||   to_room->sector_type == SECT_AIR )
	{
	    if ( !IS_AFFECTED(ch, AFF_FLYING) )
	    {
		send_to_char( "You can't fly.\r\n", ch );
		return;
	    }
	}

	if ( in_room->sector_type == SECT_WATER_NOSWIM
	||   to_room->sector_type == SECT_WATER_NOSWIM )
	{
	    OBJ_DATA *obj;
	    bool found;

	    /*
	     * Look for a boat.
	     */
	    found = FALSE;
	    for ( obj = ch->carrying; obj != NULL; obj = obj->next_content )
	    {
		if ( obj->item_type == ITEM_BOAT )
		{
		    found = TRUE;
		    break;
		}
	    }
	    if ( !found )
	    {
		send_to_char( "You need a boat to go there.\r\n", ch );
		return;
	    }
	}
    }

	//fixme limp
    if ( !IS_AFFECTED(ch, AFF_SNEAK)
    && ( IS_NPC(ch) || !IS_SET(ch->act, PLR_WIZINVIS) ) )
	{
	if (!IS_NPC(ch))
	act( "$n just went $T.", ch, NULL, dir_name[door], TO_ROOM );
	else
	{
		if( IS_SET(ch->affected_by, AFF_UNDEAD) )
		{
			if( IS_SET(ch->affected_by, AFF_NONCORP) )
				act( "$n moans and floats $T.", ch, NULL, dir_name[door], TO_ROOM );
			else
				act( "$n groans and shambles $T.", ch, NULL, dir_name[door], TO_ROOM );
		}
		else if( IS_SET(ch->act, ACT_CLAW) )
		act( "$n scampers $T.", ch, NULL, dir_name[door], TO_ROOM );
		else if( IS_SET(ch->act, ACT_BITE) )
		act( "$n lopes $T.", ch, NULL, dir_name[door], TO_ROOM );
		else if( IS_SET(ch->act, ACT_POUND) )
		act( "$n stomps $T.", ch, NULL, dir_name[door], TO_ROOM );
		else if( IS_SET(ch->act, ACT_CHARGE) )
		act( "$n trots $T.", ch, NULL, dir_name[door], TO_ROOM );
		else
		act( "$n just went $T.", ch, NULL, dir_name[door], TO_ROOM );
	}
	}

    char_from_room( ch );
    char_to_room( ch, to_room );
    if ( !IS_AFFECTED(ch, AFF_SNEAK)
    && ( IS_NPC(ch) || !IS_SET(ch->act, PLR_WIZINVIS) ) )
	act( "$n just arrived.", ch, NULL, NULL, TO_ROOM );

    do_look( ch, "auto" );


//fixme stun/rt/incap no follow
    for ( fch = in_room->people; fch != NULL; fch = fch_next )
    {
	fch_next = fch->next_in_room;
	if ( fch->master == ch && fch->position == POS_STANDING )
	{
		
		if( fch->position != POS_STANDING || fch->stun > 0 || fch->wait > 0 )
		{
		act( "You cannot follow $N.", fch, NULL, ch, TO_CHAR );	
		act( "$n cannot follow you.", fch, NULL, ch, TO_VICT );	
		return;
		}
	    act( "You follow $N.", fch, NULL, ch, TO_CHAR );
	    move_char( fch, door );
	}
    }

    return;
}



void do_north( CHAR_DATA *ch, char *argument )
{
    move_char( ch, DIR_NORTH );
    return;
}



void do_east( CHAR_DATA *ch, char *argument )
{
    move_char( ch, DIR_EAST );
    return;
}



void do_south( CHAR_DATA *ch, char *argument )
{
    move_char( ch, DIR_SOUTH );
    return;
}



void do_west( CHAR_DATA *ch, char *argument )
{
    move_char( ch, DIR_WEST );
    return;
}



void do_up( CHAR_DATA *ch, char *argument )
{
    move_char( ch, DIR_UP );
    return;
}



void do_down( CHAR_DATA *ch, char *argument )
{
    move_char( ch, DIR_DOWN );
    return;
}


void do_northeast( CHAR_DATA *ch, char *argument )
{
    move_char( ch, DIR_NORTHEAST );
    return;
}


void do_northwest( CHAR_DATA *ch, char *argument )
{
    move_char( ch, DIR_NORTHWEST );
    return;
}

void do_southeast( CHAR_DATA *ch, char *argument )
{
    move_char( ch, DIR_SOUTHEAST );
    return;
}

void do_southwest( CHAR_DATA *ch, char *argument )
{
    move_char( ch, DIR_SOUTHWEST );
    return;
}

void do_out( CHAR_DATA *ch, char *argument )
{
    move_char( ch, DIR_OUT );
    return;
}


int find_door( CHAR_DATA *ch, char *arg )
{
    EXIT_DATA *pexit;
    int door;

	 if ( !str_cmp( arg, "n" ) || !str_cmp( arg, "north" ) ) door = 0;
    else if ( !str_cmp( arg, "e" ) || !str_cmp( arg, "east"  ) ) door = 1;
    else if ( !str_cmp( arg, "s" ) || !str_cmp( arg, "south" ) ) door = 2;
    else if ( !str_cmp( arg, "w" ) || !str_cmp( arg, "west"  ) ) door = 3;
    else if ( !str_cmp( arg, "u" ) || !str_cmp( arg, "up"    ) ) door = 4;
    else if ( !str_cmp( arg, "d" ) || !str_cmp( arg, "down"  ) ) door = 5;
	else if ( !str_cmp( arg, "ne" ) || !str_cmp( arg, "northeast"  ) )
	door = 6;
    else if ( !str_cmp( arg, "nw" ) || !str_cmp( arg, "northwest"  ) )
	door = 7;
    else if ( !str_cmp( arg, "se" ) || !str_cmp( arg, "southeast"  ) )
	door = 8;
    else if ( !str_cmp( arg, "sw" ) || !str_cmp( arg, "southwest"  ) )
	door = 9;
    else if ( !str_cmp( arg, "se" ) || !str_cmp( arg, "southeast"  ) )
	door = 9;
	else if ( !str_cmp( arg, "o" ) || !str_cmp( arg, "out"  ) )
	door = 10;
    else
    {
	for ( door = 0; door <= 10; door++ )
	{
	    if ( ( pexit = ch->in_room->exit[door] ) != NULL
	    &&   IS_SET(pexit->exit_info, EX_ISDOOR)
	    &&   pexit->keyword != NULL
	    &&   is_name( arg, pexit->keyword ) )
		return door;
	}
	act( "I see no $T here.", ch, NULL, arg, TO_CHAR );
	return -1;
    }

    if ( ( pexit = ch->in_room->exit[door] ) == NULL )
    {
	act( "I see no door $T here.", ch, NULL, arg, TO_CHAR );
	return -1;
    }

    if ( !IS_SET(pexit->exit_info, EX_ISDOOR) )
    {
	send_to_char( "You can't do that.\r\n", ch );
	return -1;
    }

    return door;
}





void do_stand( CHAR_DATA *ch, char *argument )
{
	switch ( ch->position )
    {
	case POS_PRONE:
	case POS_KNEELING:
    case POS_SITTING:
	send_to_char( "You stand up.\r\n", ch );
	act( "$n stands up.", ch, NULL, NULL, TO_ROOM );
	ch->position = POS_STANDING;
	break;

	case POS_STANDING:
	send_to_char( "You are already standing.\r\n", ch );
	break;
    }

    return;
}

void do_lie( CHAR_DATA *ch, char *argument )
{
	switch ( ch->position )
    {
	case POS_STANDING:
	case POS_KNEELING:
    case POS_SITTING:
	send_to_char( "You lie down.\r\n", ch );
	act( "$n lies down.", ch, NULL, NULL, TO_ROOM );
	ch->position = POS_PRONE;
	break;

	case POS_PRONE:
	send_to_char( "You are already lying down.\r\n", ch );
	break;
    }

    return;
}


void do_rest( CHAR_DATA *ch, char *argument )
{
    return;
}

void do_sit( CHAR_DATA *ch, char *argument )
{
    switch ( ch->position )
    {
	case POS_PRONE:
	send_to_char( "You sit up.\r\n", ch );
	act( "$n sits up.", ch, NULL, NULL, TO_ROOM );
	ch->position = POS_SITTING;
	break;

	case POS_KNEELING:
    case POS_STANDING:
	send_to_char( "You sit down.\r\n", ch );
	act( "$n sits down.", ch, NULL, NULL, TO_ROOM );
	ch->position = POS_SITTING;
	break;
	
	case POS_SITTING:
	send_to_char( "You are already sitting.\r\n", ch );
	break;
    }
}

void do_kneel( CHAR_DATA *ch, char *argument )
{
    switch ( ch->position )
    {
	case POS_PRONE:
    case POS_SITTING:
	send_to_char( "You move into a kneeling position.\r\n", ch );
	act( "$n move into a kneeling position.", ch, NULL, NULL, TO_ROOM );
	ch->position = POS_KNEELING;
	break;

    case POS_STANDING:
	send_to_char( "You kneel down.\r\n", ch );
	act( "$n kneels down.", ch, NULL, NULL, TO_ROOM );
	ch->position = POS_KNEELING;
	break;
	
	case POS_KNEELING:
	send_to_char( "You are already kneeling.\r\n", ch );
	break;
    }
}


void do_sleep( CHAR_DATA *ch, char *argument )
{
	send_to_char("You try to sleep.\r\n", ch );
	
    return;
}



void do_wake( CHAR_DATA *ch, char *argument )
{
	send_to_char("You try to wake.\r\n", ch );
	
    return;
}


void do_recall( CHAR_DATA *ch, char *argument )
{
	send_to_char("You cannot remember anything.\r\n", ch );
	
    return;
}



void do_train( CHAR_DATA *ch, char *argument )
{
    return;
}

void do_knock( CHAR_DATA *ch, char *argument )
{
    OBJ_DATA *obj;
    ROOM_INDEX_DATA *to_room;
	ROOM_INDEX_DATA *orig_room = ch->in_room;
	
    if ( (obj = get_obj_list( ch, argument, ch->in_room->contents ) ) == NULL )
    {
	send_to_char("Knock what?\r\n", ch );
	return;
    }

    if ( obj->item_type != ITEM_BUILDING && obj->item_type != ITEM_DOOR )
    {
	send_to_char("You can't knock on that.\r\n", ch );
	return;
    }

    if ( (to_room = get_room_index( obj->value[0] )) == NULL )
    {
	send_to_char("But there is nothing on the other side.\r\n", ch );
	return;
    }
	
	act("You knock on $p.", ch, obj, NULL, TO_CHAR );
	act("$n knocks on $p.", ch, obj, NULL, TO_ROOM );
	
	char_from_room( ch );
	char_to_room( ch, to_room );

	if( obj->item_type == ITEM_DOOR )
	act("You hear a knocking sound coming from $p.", ch, obj, NULL, TO_ROOM );
	else
	act("You hear a knocking sound.", ch, NULL, NULL, TO_ROOM );

	char_from_room( ch );
	char_to_room( ch, orig_room );

    return;
}

void do_enter( CHAR_DATA *ch, char *argument )
{
    OBJ_DATA *obj;
    ROOM_INDEX_DATA *to_room;
	CHAR_DATA *fch;
    CHAR_DATA *fch_next;
	bool group = FALSE;
	
	//printf("do_enter: %s\r\n", argument );
	
    if ( (obj = get_obj_list( ch, argument, ch->in_room->contents ) ) == NULL )
    {
		//printf("go where?? %s\r\n", argument );
	
	//fixme support directions...
	
	send_to_char("Go where?\r\n", ch );
	return;
    }
	
	if( IS_SET( obj->value[1], CONT_CLOSED ) )
	{
	send_to_char("But it's closed.\r\n", ch );
	return;	
	}
	
	//printf("%s entering %s.\r\n", PERS(ch,ch), obj->short_descr );

    if ( obj->item_type != ITEM_PORTAL && obj->item_type != ITEM_BUILDING
		&& obj->item_type != ITEM_DOOR && obj->item_type != ITEM_CLIMB && obj->item_type != ITEM_SWIM)
    {
	send_to_char("You can't go there.\r\n", ch );
	return;
    }

    if ( (to_room = get_room_index( obj->value[0] )) == NULL )
    {
	send_to_char("It doesn't appear to lead anywhere.\r\n", ch );
	return;
    }

	switch( obj->item_type )
    {
	case ITEM_PORTAL:
		act("$n enters $p.", ch, obj, NULL, TO_ROOM );
	    break;
	case ITEM_BUILDING:
		act("$n goes inside $p.", ch, obj, NULL, TO_ROOM );
	    break;
	case ITEM_DOOR:
	    act("$n goes through $p.", ch, obj, NULL, TO_ROOM );
	    break;
	case ITEM_CLIMB:
		act("$n climbs over $p.", ch, NULL, NULL, TO_ROOM);
		break;
	case ITEM_SWIM:
		act("$n swims from $p.", ch, NULL, NULL, TO_ROOM);
		break;
	default:
		break;
    }
	

	for ( fch = ch->in_room->people; fch != NULL; fch = fch_next )
    {
	fch_next = fch->next_in_room;
	if ( fch->master == ch && fch->position == POS_STANDING )
	{

		
		if( fch->position != POS_STANDING || fch->stun > 0 || fch->wait > 0 )
		{
		act( "You cannot follow $N.", fch, NULL, ch, TO_CHAR );	
		act( "$n cannot follow you.", fch, NULL, ch, TO_VICT );	
		return;
		}
		
		//duplicative and spammy given messaging further below.
		//act( "You follow $N.", fch, NULL, ch, TO_CHAR );
		
	    char_from_room( fch );
		char_to_room( fch, to_room );
		group = TRUE;
	}
    }
	
	char_from_room( ch );
	char_to_room( ch, to_room );

	//fixme climb/swim group??
	//NOTVICT??
	if( group )
	{
    switch( obj->item_type )
    {
	case ITEM_PORTAL:
	    act("$n's group arrives from $p.", ch, obj, NULL, TO_ROOM );
	    break;
	case ITEM_BUILDING:
		act("$n's group arrives from outside.", ch, NULL, NULL, TO_ROOM);
		break;
	case ITEM_DOOR:
	    act("$n's group comes through $p.", ch, obj, NULL, TO_ROOM );
	    break;
	case ITEM_CLIMB:
		act("$n's group climbs over $p.", ch, NULL, NULL, TO_ROOM);
		break;
	case ITEM_SWIM:
		act("$n's group swims towards $p.", ch, NULL, NULL, TO_ROOM);
		break;
	default:
	    break;
    }
	}	
	
	else
	{
    switch( obj->item_type )
    {
	case ITEM_PORTAL:
	    act("$n arrives from $p.", ch, obj, NULL, TO_ROOM );
	    break;
	case ITEM_BUILDING:
		act("$n arrives from outside.", ch, NULL, NULL, TO_ROOM);
		break;
	case ITEM_DOOR:
	    act("$n comes through $p.", ch, obj, NULL, TO_ROOM );
	    break;
	case ITEM_CLIMB:
		act("$n climbs over $p.", ch, NULL, NULL, TO_ROOM);
		break;
	case ITEM_SWIM:
		act("$n swims towards $p.", ch, NULL, NULL, TO_ROOM);
		break;
	default:
	    break;
    }
	}
    do_look( ch, "" );
	
	for ( fch = ch->in_room->people; fch != NULL; fch = fch_next )
    {
	fch_next = fch->next_in_room;
	if ( fch->master == ch )
		do_look( fch, "" );
	}
	
    return;
}


#if defined(macintosh)
#include <types.h>
#else
#include <sys/types.h>
#endif
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <time.h>
#include "merc.h"



/*
 * Local functions.
 */
#define CD CHAR_DATA
bool	remove_obj	args( ( CHAR_DATA *ch,  OBJ_DATA *obj ) );
void	wear_obj	args( ( CHAR_DATA *ch, OBJ_DATA *obj, bool fReplace ) );
CD *	find_keeper	args( ( CHAR_DATA *ch ) );
int	get_cost	args( ( CHAR_DATA *keeper, OBJ_DATA *obj, bool fBuy ) );
#undef	CD



//fixme
void get_obj( CHAR_DATA *ch, OBJ_DATA *obj, OBJ_DATA *container )
{
    if ( !CAN_WEAR(obj, ITEM_EQUIP_TAKE) )
    {
	send_to_char( "You cannot take that.\r\n", ch );
	return;
    }
	
	if( obj->item_type != ITEM_MONEY )
	{
	if( get_eq_char( ch, EQUIP_RIGHTHAND ) != NULL &&  get_eq_char( ch, EQUIP_LEFTHAND ) != NULL )
    {
	send_to_char( "Your hands are full.\r\n", ch );
	return;
    }
	}
	
	/*
    if ( ch->carry_number + get_obj_number( obj ) > can_carry_n( ch ) )
    {
	act( "$d: you can't carry that many items.",
	    ch, NULL, obj->name, TO_CHAR );
	return;
    }

    if ( ch->carry_weight + get_obj_weight( obj ) > can_carry_w( ch ) )
    {
	act( "$d: you can't carry that much weight.",
	    ch, NULL, obj->name, TO_CHAR );
	return;
    }
	*/

    if ( container != NULL )
    {
		
	if( container->carried_by != NULL && container->carried_by == ch )
	{
		act( "You remove $p from your $O.", ch, obj, container, TO_CHAR );
		act( "$n removes $p from $s $O.", ch, obj, container, TO_ROOM );
	}
	else
	{
		act( "You remove $p from $P.", ch, obj, container, TO_CHAR );
		act( "$n removes $p from $P.", ch, obj, container, TO_ROOM );
	}
	obj_from_obj( obj );
    }
    else
    {
	act( "You pick up $p.", ch, obj, container, TO_CHAR );
	act( "$n picks up $p.", ch, obj, container, TO_ROOM );
	obj_from_room( obj );
    }

    if ( obj->item_type == ITEM_MONEY )
    {
	if( obj->value[0] > 1 )
	printf_to_char( ch, "There were %d silver coins.\r\n", obj->value[0] );
	
	ch->gold += obj->value[0];
	extract_obj( obj );
    }
    else
    {
	obj_to_char( obj, ch );
	obj_to_free_hand( obj, ch );
    }

    return;
}




void do_get( CHAR_DATA *ch, char *argument )
{
    char arg1[MAX_INPUT_LENGTH];
    char arg2[MAX_INPUT_LENGTH];
    OBJ_DATA *obj;
    OBJ_DATA *container;
	bool fromContainer = FALSE;
	
    argument = one_argument( argument, arg1 );
    argument = one_argument( argument, arg2 );

    /* Get type. */
    if ( arg1[0] == '\0' )
    {
	send_to_char( "Get what?\r\n", ch );
	return;
    }

	// 1 arg.
    if ( arg2[0] == '\0' )
    {
	    /* 'get obj' */			
		obj = get_obj_list( ch, arg1, ch->in_room->contents );
		
		if( obj == NULL )
		{
		obj = get_obj_in_room_containers( ch, arg1 ); // can optimize by passing container here.
		
		if( obj != NULL )
		{
			fromContainer = TRUE;
			container = get_obj_in_room_containers2( ch, arg1 ); 
		}
		}
		
		if( obj == NULL )
		{
		obj = get_obj_in_carried_containers( ch, arg1 );  // can optimize by passing container here.
		
		if( obj != NULL )
		{
			fromContainer = TRUE;
			container = get_obj_in_carried_containers2( ch, arg1 );
		}
		}
		
		
		if( obj == NULL )
		obj = get_obj_wear( ch, arg1 );

	    if ( obj == NULL )
	    {
		send_to_char( "Get what?\r\n", ch );
		return;
	    }
		
		if( obj->wear_loc != EQUIP_NONE )
		{
		printf_to_char( ch, "You already have that.\r\n" );
		return;
		}

		if( fromContainer == TRUE )
		{
			if( container->item_type == ITEM_MERCHANT_TABLE )
			{
			printf_to_char( ch, "You inquire about the price of %s.  You can BUY it for %d silver coins.\r\n", obj->short_descr, obj->cost );
			act( "$n inquires about the price of $p on $P.", ch, obj, container, TO_ROOM );	
			return;
			}
			else
				get_obj( ch, obj, container );	
		}
		else
			get_obj( ch, obj, NULL );
    }
    else
    {
	/* 'get ... container' */

	if ( ( container = get_obj_here( ch, arg2 ) ) == NULL )
	{
	    //act( "I see no $T here.", ch, NULL, arg2, TO_CHAR );
		send_to_char("Get from what?\r\n", ch);
	    return;
	}
	
	if (!( container->item_type == ITEM_CONTAINER || container->item_type == ITEM_FURNITURE || container->item_type == ITEM_MERCHANT_TABLE ))
    {
	send_to_char( "It really could not hold anything.\r\n", ch );
	return;
    }

	if ( IS_SET(container->value[1], CONT_CLOSED) || IS_SET( container->value[1], CONT_LOCKED ) )
	{
	    //act( "The $d is closed.", ch, NULL, container->name, TO_CHAR );
		send_to_char("It is closed.\r\n", ch);
	    return;
	}

	    //'get obj container'
	    obj = get_obj_list( ch, arg1, container->contains );
	    if ( obj == NULL )
	    {
		//act( "I see nothing like that in the $T.",
		//    ch, NULL, arg2, TO_CHAR );
		send_to_char("Get what from what?\r\n", ch);
		return;
	    }
		
		if( container->item_type == ITEM_MERCHANT_TABLE )
		{
			printf_to_char( ch, "You inquire about the price of %s.  You can BUY it for %d silver coins.\r\n", obj->short_descr, obj->cost );
			act( "$n inquires about the price of $p on $P.", ch, obj, container, TO_ROOM );

			return;
		}
		else
			get_obj( ch, obj, container );
    }

    return;
}



void do_put( CHAR_DATA *ch, char *argument )
{
    char arg1[MAX_INPUT_LENGTH];
    char arg2[MAX_INPUT_LENGTH];
    OBJ_DATA *container;
    OBJ_DATA *obj;
    OBJ_DATA *obj_next;

    argument = one_argument( argument, arg1 );
    argument = one_argument( argument, arg2 );

    if ( arg1[0] == '\0' || arg2[0] == '\0' )
    {
	send_to_char( "Put what where?\r\n", ch );
	return;
    }
	
    if ( !str_cmp( arg2, "all" ) || !str_prefix( "all.", arg2 ) )
    {
	send_to_char( "That does not work.\r\n", ch );
	return;
    }

    if ( ( container = get_obj_here( ch, arg2 ) ) == NULL )
    {
	//act( "I see no $T here.", ch, NULL, arg2, TO_CHAR );
	send_to_char( "Put it where?\r\n", ch );
	return;
    }
	
    if (!( container->item_type == ITEM_CONTAINER || container->item_type == ITEM_FURNITURE || container->item_type == ITEM_MERCHANT_TABLE ))
    {
	send_to_char( "It really could not hold anything.\r\n", ch );
	return;
    }
	
	// this allows imms to set up traveling merchants, etc.
	if ( !IS_IMMORTAL(ch) && container->item_type == ITEM_MERCHANT_TABLE )
	{
	send_to_char( "They don't accept consignment here.\r\n", ch );
	return;
    }	

    if ( IS_SET(container->value[1], CONT_CLOSED) )
    {
	//act( "The $d is closed.", ch, NULL, container->name, TO_CHAR );
	send_to_char( "It is closed.\r\n", ch );
	return;
    }

    if ( str_cmp( arg1, "all" ) && str_prefix( "all.", arg1 ) )
    {
	/* 'put obj container' */
	if ( ( obj = get_obj_carry( ch, arg1 ) ) == NULL )
	{
	    send_to_char( "Put what?\r\n", ch );
	    return;
	}

	if ( obj == container )
	{
	    send_to_char( "It cannot hold itself.\r\n", ch );
	    return;
	}

	if ( !can_drop_obj( ch, obj ) )
	{
	    send_to_char( "It seems to be stuck to you.\r\n", ch );
	    return;
	}

	if ( get_obj_weight( obj ) + get_obj_weight( container )
	     > container->value[0] )
	{
	    send_to_char( "It cannot hold that much.\r\n", ch );
	    return;
	}

	obj_from_char( obj );
	obj_to_obj( obj, container );
	
	if( container->item_type == ITEM_CONTAINER )
	{
	if( container->carried_by != NULL && container->carried_by == ch )
	{
	act( "$n puts $p in $P.", ch, obj, container, TO_ROOM );
	act( "You put $p in $P.", ch, obj, container, TO_CHAR );
	}	
	else
	{
	act( "$n puts $p in $s $O.", ch, obj, container, TO_ROOM );
	act( "You put $p in your $O.", ch, obj, container, TO_CHAR );
	}
	}
	else
	{
	act( "$n puts $p on $P.", ch, obj, container, TO_ROOM );
	act( "You put $p on $P.", ch, obj, container, TO_CHAR );	
	}
    }
    else
    {
	/* 'put all container' or 'put all.obj container' */
	for ( obj = ch->carrying; obj != NULL; obj = obj_next )
	{
	    obj_next = obj->next_content;

	    if ( ( arg1[3] == '\0' || is_name( &arg1[4], obj->name ) )
	    &&   can_see_obj( ch, obj )
	    &&   obj->wear_loc == EQUIP_NONE
	    &&   obj != container
	    &&   can_drop_obj( ch, obj )
	    &&   get_obj_weight( obj ) + get_obj_weight( container )
		 <= container->value[0] )
	    {
		obj_from_char( obj );
		obj_to_obj( obj, container );
		act( "$n puts $p in $P.", ch, obj, container, TO_ROOM );
		act( "You put $p in $P.", ch, obj, container, TO_CHAR );
	    }
	}
    }

    return;
}

bool obj_on_floor( CHAR_DATA *ch, OBJ_DATA *obj )
{
    OBJ_DATA *r;
	
	if( obj == NULL || ch->in_room == NULL )
		return FALSE;

    for ( r = ch->in_room->contents; r != NULL; r = r->next_content )
    {
	if( r == obj )
	return TRUE;
    }

    return FALSE;
}



void do_newput( CHAR_DATA *ch, char *argument )
{
    char arg1[MAX_INPUT_LENGTH];
    char arg2[MAX_INPUT_LENGTH];
    OBJ_DATA *container;
    OBJ_DATA *obj = NULL;
    OBJ_DATA *obj_next;
	
    argument = one_argument( argument, arg1 );
    argument = one_argument( argument, arg2 );

    if ( arg1[0] == '\0' )
    {
	send_to_char( "Put what where?\r\n", ch );
	return;
    }
	
	if ( arg2[0] == '\0' )
    {
	do_drop( ch, arg1 );
	return;
    }
	
    if ( !str_cmp( arg2, "all" ) || !str_prefix( "all.", arg2 ) )
    {
	send_to_char( "That does not work.\r\n", ch );
	return;
    }

    if ( ( container = get_obj_here( ch, arg2 ) ) == NULL )
    {
	//act( "I see no $T here.", ch, NULL, arg2, TO_CHAR );
	send_to_char( "Put it where?\r\n", ch );
	return;
    }
	
	if( (container->carried_by == NULL || container->carried_by != ch ) && !obj_on_floor( ch, container ) )
	{
		printf_to_char( ch, "Take out %s first.\r\n", container->short_descr );
		return;
	}

    if (!( container->item_type == ITEM_CONTAINER || container->item_type == ITEM_FURNITURE || container->item_type == ITEM_MERCHANT_TABLE ))
    {
	send_to_char( "It really could not hold anything.\r\n", ch );
	return;
    }
	
	// this allows imms to set up traveling merchants, etc.
	if ( !IS_IMMORTAL(ch) && container->item_type == ITEM_MERCHANT_TABLE )
	{
	send_to_char( "They don't accept consignment here.\r\n", ch );
	return;
    }	

    if ( IS_SET(container->value[1], CONT_CLOSED) )
    {
	//act( "The $d is closed.", ch, NULL, container->name, TO_CHAR );
	send_to_char( "It is closed.\r\n", ch );
	return;
    }

    if ( str_cmp( arg1, "all" ) && str_prefix( "all.", arg1 ) )
    {
	/* 'put obj container' */
	if ( ( obj = get_obj_carry( ch, arg1 ) ) == NULL )
	{
	    send_to_char( "Put what?\r\n", ch );
	    return;
	}

	if ( obj == container )
	{
	    send_to_char( "It cannot hold itself.\r\n", ch );
	    return;
	}

	if ( !can_drop_obj( ch, obj ) )
	{
	    send_to_char( "It seems to be stuck to you.\r\n", ch );
	    return;
	}

	if ( get_obj_weight( obj ) + get_obj_weight( container )
	     > container->value[0] )
	{
	    send_to_char( "It cannot hold that much.\r\n", ch );
	    return;
	}

	obj_from_char( obj );
	obj_to_obj( obj, container );
	
	if( container->item_type == ITEM_CONTAINER )
	{
	if( container->carried_by != NULL && container->carried_by == ch )
	{
	act( "$n puts $p in $P.", ch, obj, container, TO_ROOM );
	act( "You put $p in $P.", ch, obj, container, TO_CHAR );
	}	
	else
	{
	act( "$n puts $p in $s $O.", ch, obj, container, TO_ROOM );
	act( "You put $p in your $O.", ch, obj, container, TO_CHAR );
	}
	}
	else
	{
	act( "$n puts $p on $P.", ch, obj, container, TO_ROOM );
	act( "You put $p on $P.", ch, obj, container, TO_CHAR );	
	}
    }
    else
    {
	/* 'put all container' or 'put all.obj container' */
	for ( obj = ch->carrying; obj != NULL; obj = obj_next )
	{
	    obj_next = obj->next_content;

	    if ( ( arg1[3] == '\0' || is_name( &arg1[4], obj->name ) )
	    &&   can_see_obj( ch, obj )
	    &&   obj->wear_loc == EQUIP_NONE
	    &&   obj != container
	    &&   can_drop_obj( ch, obj )
	    &&   get_obj_weight( obj ) + get_obj_weight( container )
		 <= container->value[0] )
	    {
		obj_from_char( obj );
		obj_to_obj( obj, container );
		act( "$n puts $p in $P.", ch, obj, container, TO_ROOM );
		act( "You put $p in $P.", ch, obj, container, TO_CHAR );
	    }
	}
    }

    return;
}




void do_drop( CHAR_DATA *ch, char *argument )
{
    char arg[MAX_INPUT_LENGTH];
    OBJ_DATA *obj;
    OBJ_DATA *obj_next;
    bool found;

    argument = one_argument( argument, arg );

    if ( arg[0] == '\0' )
    {
	send_to_char( "Drop what?\r\n", ch );
	return;
    }

    if ( is_number( arg ) )
    {
	/* 'drop NNNN coins' */
	int amount;

	amount   = atoi(arg);
	argument = one_argument( argument, arg );
	if ( amount <= 0
	|| ( str_cmp( arg, "coins" ) && str_cmp( arg, "coin" ) && str_cmp( arg, "silver" ) ) )
	{
	    send_to_char( "Sorry, you can't do that.\r\n", ch );
	    return;
	}

	if ( ch->gold < amount )
	{
	    send_to_char( "You haven't got that many coins.\r\n", ch );
	    return;
	}

	ch->gold -= amount;
	
	if( amount > 1 )
	act( "$n drops some coins.", ch, NULL, NULL, TO_ROOM );
	else
	act( "$n drops a coin.", ch, NULL, NULL, TO_ROOM );
	
	printf_to_char( ch, "You drop %d coin%s.\r\n", amount, amount>1?"s":"");

	for ( obj = ch->in_room->contents; obj != NULL; obj = obj_next )
	{
	    obj_next = obj->next_content;

	    switch ( obj->pIndexData->vnum )
	    {
	    case OBJ_VNUM_MONEY_ONE:
		amount += 1;
		extract_obj( obj );
		break;

	    case OBJ_VNUM_MONEY_SOME:
		amount += obj->value[0];
		extract_obj( obj );
		break;
	    }
	}

	obj_to_room( create_money( amount ), ch->in_room );
	
	return;
    }

    if ( str_cmp( arg, "all" ) && str_prefix( "all.", arg ) )
    {
	/* 'drop obj' */
	if ( ( obj = get_obj_carry( ch, arg ) ) == NULL )
	{
	    send_to_char( "You do not have that item.\r\n", ch );
	    return;
	}

	if ( !can_drop_obj( ch, obj ) )
	{
	    send_to_char( "As you try to drop it, it glows with a dark aura and sticks to your hand!\r\n", ch );
	    return;
	}

	obj_from_char( obj );
	obj_to_room( obj, ch->in_room );
	act( "$n drops $p.", ch, obj, NULL, TO_ROOM );
	act( "You drop $p.", ch, obj, NULL, TO_CHAR );
    }
    else
    {
	/* 'drop all' or 'drop all.obj' */
	found = FALSE;
	for ( obj = ch->carrying; obj != NULL; obj = obj_next )
	{
	    obj_next = obj->next_content;

	    if ( ( arg[3] == '\0' || is_name( &arg[4], obj->name ) )
	    &&   can_see_obj( ch, obj )
	    &&   obj->wear_loc == EQUIP_NONE
	    &&   can_drop_obj( ch, obj ) )
	    {
		found = TRUE;
		obj_from_char( obj );
		obj_to_room( obj, ch->in_room );
		act( "$n drops $p.", ch, obj, NULL, TO_ROOM );
		act( "You drop $p.", ch, obj, NULL, TO_CHAR );
	    }
	}

	if ( !found )
	{
	    if ( arg[3] == '\0' )
		act( "You are not carrying anything.",
		    ch, NULL, arg, TO_CHAR );
	    else
		act( "You are not carrying any $T.",
		    ch, NULL, &arg[4], TO_CHAR );
	}
    }

    return;
}



void do_show( CHAR_DATA *ch, char *argument )
{
    char arg1 [MAX_INPUT_LENGTH];
    char arg2 [MAX_INPUT_LENGTH];
    CHAR_DATA *victim;
    OBJ_DATA  *obj;

    argument = one_argument( argument, arg1 );
    argument = one_argument( argument, arg2 );

    if ( arg1[0] == '\0' || arg2[0] == '\0' )
    {
	send_to_char( "Show what to whom?\r\n", ch );
	return;
    }

    if ( ( obj = get_obj_carry( ch, arg1 ) ) == NULL )
    {
	send_to_char( "You do not have that item.\r\n", ch );
	return;
    }

    if ( ( victim = get_char_room( ch, arg2 ) ) == NULL )
    {
	send_to_char( "They aren't here.\r\n", ch );
	return;
	}
	
    if ( !can_see_obj( victim, obj ) )
    {
	act( "$N can't see it.", ch, NULL, victim, TO_CHAR );
	return;
    }

    act( "$n shows $p to $N.", ch, obj, victim, TO_NOTVICT );
    act( "$n shows you $p.",   ch, obj, victim, TO_VICT    );
    act( "You show $p to $N.", ch, obj, victim, TO_CHAR    );
	
	if( obj->description )
	send_to_char( obj->description, victim );
	else
	send_to_char( "You notice nothing unusual about it.\r\n", victim );
    
	return;
}


void do_give( CHAR_DATA *ch, char *argument )
{
    char arg1 [MAX_INPUT_LENGTH];
    char arg2 [MAX_INPUT_LENGTH];
    CHAR_DATA *victim;
    OBJ_DATA  *obj;

    argument = one_argument( argument, arg1 );
    argument = one_argument( argument, arg2 );

    if ( arg1[0] == '\0' || arg2[0] == '\0' )
    {
	send_to_char( "Give what to whom?\r\n", ch );
	return;
    }

    if ( is_number( arg1 ) )
    {
	/* 'give NNNN coins victim' */
	int amount;

	amount   = atoi(arg1);
	if ( amount <= 0
	|| ( str_cmp( arg2, "coins" ) && str_cmp( arg2, "coin" ) ) )
	{
	    send_to_char( "Sorry, you can't do that.\r\n", ch );
	    return;
	}

	argument = one_argument( argument, arg2 );
	if ( arg2[0] == '\0' )
	{
	    send_to_char( "Give what to whom?\r\n", ch );
	    return;
	}

	if ( ( victim = get_char_room( ch, arg2 ) ) == NULL )
	{
	    send_to_char( "They aren't here.\r\n", ch );
	    return;
	}

	if ( ch->gold < amount )
	{
	    send_to_char( "You haven't got that much silver.\r\n", ch );
	    return;
	}

	ch->gold     -= amount;
	victim->gold += amount;
	act( "$n gives $N some silver.",  ch, NULL, victim, TO_NOTVICT );
	
	printf_to_char( ch, "You give %s %d silver coins.\r\n", PERS(victim,ch), amount );
	printf_to_char( victim, "%s gives you %d silver coins.\r\n", PERS(ch,victim), amount );
	return;
    }

    if ( ( obj = get_obj_carry( ch, arg1 ) ) == NULL )
    {
	send_to_char( "You do not have that item.\r\n", ch );
	return;
    }

    if ( obj->wear_loc != EQUIP_NONE && obj->wear_loc != EQUIP_RIGHTHAND && obj->wear_loc != EQUIP_LEFTHAND )
    {
	send_to_char( "You must remove it first.\r\n", ch );
	return;
    }

    if ( ( victim = get_char_room( ch, arg2 ) ) == NULL )
    {
	send_to_char( "They aren't here.\r\n", ch );
	return;
    }

    if ( !can_drop_obj( ch, obj ) )
    {
	send_to_char( "You can't let go of it.\r\n", ch );
	return;
    }
	
	if( get_eq_char( victim, EQUIP_RIGHTHAND ) != NULL &&  get_eq_char( victim, EQUIP_LEFTHAND ) != NULL )
    {
	send_to_char( "Their hands are full.\r\n", ch );
	return;
    }

    if ( victim->carry_number + get_obj_number( obj ) > can_carry_n( victim ) )
    {
	act( "$N has $S hands full.", ch, NULL, victim, TO_CHAR );
	return;
    }

    if ( victim->carry_weight + get_obj_weight( obj ) > can_carry_w( victim ) )
    {
	act( "$N can't carry that much weight.", ch, NULL, victim, TO_CHAR );
	return;
    }

    if ( !can_see_obj( victim, obj ) )
    {
	act( "$N can't see it.", ch, NULL, victim, TO_CHAR );
	return;
    }

    obj_from_char( obj );
    obj_to_char( obj, victim );
	obj_to_free_hand( obj, victim );
	
    act( "$n gives $p to $N.", ch, obj, victim, TO_NOTVICT );
    act( "$n gives you $p.",   ch, obj, victim, TO_VICT    );
    act( "You give $p to $N.", ch, obj, victim, TO_CHAR    );
    return;
}




void do_fill( CHAR_DATA *ch, char *argument )
{
    char arg[MAX_INPUT_LENGTH];
    OBJ_DATA *obj;
    OBJ_DATA *fountain;
    bool found;

    one_argument( argument, arg );

    if ( arg[0] == '\0' )
    {
	send_to_char( "Fill what?\r\n", ch );
	return;
    }

    if ( ( obj = get_obj_carry( ch, arg ) ) == NULL )
    {
	send_to_char( "You do not have that item.\r\n", ch );
	return;
    }

    found = FALSE;
    for ( fountain = ch->in_room->contents; fountain != NULL;
	fountain = fountain->next_content )
    {
	if ( fountain->item_type == ITEM_FOUNTAIN )
	{
	    found = TRUE;
	    break;
	}
    }

    if ( !found )
    {
	send_to_char( "There is no fountain here!\r\n", ch );
	return;
    }

    if ( obj->item_type != ITEM_DRINK_CON )
    {
	send_to_char( "You can't fill that.\r\n", ch );
	return;
    }

    if ( obj->value[1] != 0 && obj->value[2] != 0 )
    {
	send_to_char( "There is already another liquid in it.\r\n", ch );
	return;
    }

    if ( obj->value[1] >= obj->value[0] )
    {
	send_to_char( "Your container is full.\r\n", ch );
	return;
    }

    act( "You fill $p.", ch, obj, NULL, TO_CHAR );
    obj->value[2] = 0;
    obj->value[1] = obj->value[0];
    return;
}



void do_drink( CHAR_DATA *ch, char *argument )
{
    char arg[MAX_INPUT_LENGTH];
    OBJ_DATA *obj;

    one_argument( argument, arg );

    if ( arg[0] == '\0' )
    {
	for ( obj = ch->in_room->contents; obj; obj = obj->next_content )
	{
	    if ( obj->item_type == ITEM_FOUNTAIN )
		break;
	}

	if ( obj == NULL )
	{
	    send_to_char( "Drink what?\r\n", ch );
	    return;
	}
	
	act( "$n drinks from the fountain.", ch, NULL, NULL, TO_ROOM );
	send_to_char( "You drink from the fountain.\r\n", ch );
	
	if( number_bits(2) )
	send_to_char( "Refreshing.\r\n", ch );
	return;
    }
	
	
    else
    {
	if ( ( obj = get_obj_close( ch, arg ) ) == NULL )
	{
		
		if( is_name(arg, "fountain") )
		do_drink(ch, "");
		else
	    send_to_char( "You can't find it.\r\n", ch );
	    return;
	}
	else
	{
	
	switch ( obj->item_type )
    {
	case ITEM_POTION:
	printf_to_char( ch, "You drink your %s.\r\n", smash_article(obj->short_descr) );
    act( "$n drinks $o.", ch, obj, NULL, TO_ROOM );

	if( get_gsn(obj->value[0]) == -1 )
	{
		send_to_char( "The potion tastes gritty and flavorless.\r\n", ch );
		bug( "potion - trying to cast spell %d", obj->value[0] );
		return;
	}
	
    obj_cast_spell( get_gsn(obj->value[0]), 1, ch, ch, NULL );
    
    extract_obj( obj );
	
	/*
	obj = create_object( get_obj_index( OBJ_VNUM_POTION_FLASK ), 0 );
	obj_to_char( obj, ch );
	obj_to_free_hand( obj, ch );
	*/
	
	return;

    case ITEM_FOUNTAIN:
	act( "$n drinks from the fountain.", ch, NULL, NULL, TO_ROOM );
	send_to_char( "You drink from the fountain.\r\n", ch );
	
	if( number_bits(2) )
	send_to_char( "Refreshing.\r\n", ch );
	break;

    case ITEM_DRINK_CON:
	if ( obj->value[1] <= 0 )
	{
	    send_to_char( "It is already empty.\r\n", ch );
	    return;
		
	default:
	printf_to_char( ch, "You can't drink from a %s!\r\n", smash_article(obj->short_descr) );
	break;
	}

	act( "$n takes a drink from $p.",
	    ch, obj, NULL, TO_ROOM );
	act( "You take a drink from $p.",
	    ch, obj, NULL, TO_CHAR );
	
	if( number_bits(2) )
	send_to_char( "Refreshing.\r\n", ch );

	obj->value[1] -= 1;
	if ( obj->value[1] <= 0 )
	{
	    send_to_char( "That was the last of it.\r\n", ch );
	    extract_obj( obj );
	}
	break;
    }
	
	}
	
    }
	

    return;
}



void do_eat( CHAR_DATA *ch, char *argument )
{
    char arg[MAX_INPUT_LENGTH];
    OBJ_DATA *obj;

    one_argument( argument, arg );
    if ( arg[0] == '\0' )
    {
	send_to_char( "Eat what?\r\n", ch );
	return;
    }

    if ( ( obj = get_obj_carry( ch, arg ) ) == NULL )
    {
	send_to_char( "You do not have that item.\r\n", ch );
	return;
    }

    if ( !IS_IMMORTAL(ch) )
    {
	if ( obj->item_type != ITEM_FOOD && obj->item_type != ITEM_PILL )
	{
	    send_to_char( "That's not edible.\r\n", ch );
	    return;
	}
/*
	if ( !IS_NPC(ch) && ch->pcdata->condition[COND_FULL] > 40 )
	{
	    send_to_char( "You are too full to eat more.\r\n", ch );
	    return;
	}
*/
    }

    act( "$n eats $p.",  ch, obj, NULL, TO_ROOM );
    act( "You eat $p.", ch, obj, NULL, TO_CHAR );
	
	if( obj->item_type == ITEM_FOOD )
	{
	switch( number_range(0, 9 ) )
	{
	case 0:
	send_to_char( "Tasty.\r\n", ch );
	break;
	case 1:
	send_to_char( "Yum.\r\n", ch );
	break;
	case 2:
	send_to_char( "Just like mom used to make.\r\n", ch );
	break;
	default:
	break;
	}
	}
	
    switch ( obj->item_type )
    {

    case ITEM_FOOD:
	/*
	if ( !IS_NPC(ch) )
	{
	    int condition;

	    condition = ch->pcdata->condition[COND_FULL];
	    gain_condition( ch, COND_FULL, obj->value[0] );
	    if ( condition == 0 && ch->pcdata->condition[COND_FULL] > 0 )
		send_to_char( "You are no longer hungry.\r\n", ch );
	    else if ( ch->pcdata->condition[COND_FULL] > 40 )
		send_to_char( "You are full.\r\n", ch );
	}

	if ( obj->value[3] != 0 )
	{
	    AFFECT_DATA af;

	    act( "$n chokes and gags.", ch, 0, 0, TO_ROOM );
	    send_to_char( "You choke and gag.\r\n", ch );

	    af.type      = gsn_poison;
	    af.duration  = 2 * obj->value[0];
	    af.location  = APPLY_NONE;
	    af.modifier  = 0;
	    af.bitvector = AFF_POISON;
	    affect_join( ch, &af );
	}
	*/
	break;

    case ITEM_PILL: //fixme..
	obj_cast_spell( obj->value[1], obj->value[0], ch, ch, NULL );
	obj_cast_spell( obj->value[2], obj->value[0], ch, ch, NULL );
	obj_cast_spell( obj->value[3], obj->value[0], ch, ch, NULL );
	break;
    }

    extract_obj( obj );
    return;
}



/*
 * Remove an object.
 */
bool remove_obj( CHAR_DATA *ch, OBJ_DATA *obj )
{
    if ( obj == NULL )
		return TRUE;
	
	if( obj->wear_loc == EQUIP_RIGHTHAND || obj->wear_loc == EQUIP_LEFTHAND )
		return TRUE;

    if ( IS_SET(obj->extra_flags, ITEM_NOREMOVE) )
    {
		act( "You cannot remove your $o.", ch, obj, NULL, TO_CHAR );
		return FALSE;
    }

    unequip_char( ch, obj );
	
	if( obj->wear_loc == EQUIP_SHOULDER ) // && obj->item_type == ITEM_SHIELD )
	{
    act( "$n slings $s $o off $s shoulder.", ch, obj, NULL, TO_ROOM );
    act( "You sling your $o off your shoulder.", ch, obj, NULL, TO_CHAR );
    }
	else
	{
	act( "$n removes $s $o.", ch, obj, NULL, TO_ROOM );
    act( "You remove your $o.", ch, obj, NULL, TO_CHAR );
    }
	
	return TRUE;
}

int get_num_equipped( CHAR_DATA *ch, int iWear )
{
	OBJ_DATA *obj;
    int num = 0;

    for ( obj = ch->carrying; obj != NULL; obj = obj->next_content )
    {
	if ( obj->wear_loc == iWear )
	    num++;
    }
	
	//free_obj( obj );

    return num;
}

int new_get_max_worn( int iWear )
{
	switch ( iWear )
	{
case  EQUIP_RIGHTHAND: return 1;
case  EQUIP_LEFTHAND: return 1;
case  EQUIP_HEAD: return 1;
case  EQUIP_EARS: return 2;
case  EQUIP_NECK: return 1;
case  EQUIP_AROUND_NECK: return 3;
case  EQUIP_BACK: return 1;
case  EQUIP_ARMS: return 1;
case  EQUIP_HANDS: return 1;
case  EQUIP_WRIST: return 2;
case  EQUIP_FRONT: return 1;
case  EQUIP_CHEST: return 1;
case  EQUIP_BELT: return 1;
case  EQUIP_ONBELT: return 3;
case  EQUIP_LEGS: return 1;
case  EQUIP_ANKLE: return 1;
case  EQUIP_FEET: return 1;
case  EQUIP_PIN: return 8;
case  EQUIP_SHOULDER: return 1;
case  EQUIP_CLOAK: return 1;
case  EQUIP_FINGER: return 2;
case EQUIP_NONE:
	default: return 0;
	}
}

int new_get_wear_loc( OBJ_DATA *obj )
{	
	if( CAN_WEAR( obj, ITEM_EQUIP_FINGER ) )
		return EQUIP_FINGER;
		
	else if( CAN_WEAR( obj, ITEM_EQUIP_NECK ) )
		return EQUIP_NECK;
		
	else if( CAN_WEAR( obj, ITEM_EQUIP_AROUND_NECK ) )
		return EQUIP_AROUND_NECK;
		
	else if( CAN_WEAR( obj, ITEM_EQUIP_CHEST ) )
		return EQUIP_CHEST;
		
	else if( CAN_WEAR( obj, ITEM_EQUIP_HEAD ) )
		return EQUIP_HEAD;
		
	else if( CAN_WEAR( obj, ITEM_EQUIP_ONBELT ) )
		return EQUIP_ONBELT;
		
	else if( CAN_WEAR( obj, ITEM_EQUIP_LEGS ) )
		return EQUIP_LEGS;
		
	else if( CAN_WEAR( obj, ITEM_EQUIP_ANKLE ) )
		return EQUIP_ANKLE;
		
	else if( CAN_WEAR( obj, ITEM_EQUIP_FEET ) )
		return EQUIP_FEET;
		
	else if( CAN_WEAR( obj, ITEM_EQUIP_HANDS ) )
		return EQUIP_HANDS;
		
	else if( CAN_WEAR( obj, ITEM_EQUIP_ARMS ) )
		return EQUIP_ARMS;
		
	else if( CAN_WEAR( obj, ITEM_EQUIP_FRONT ) )
		return EQUIP_FRONT;
		
	else if( CAN_WEAR( obj, ITEM_EQUIP_BELT ) )
		return EQUIP_BELT;
	
		
	else if( CAN_WEAR( obj, ITEM_EQUIP_WRIST ) )
		return EQUIP_WRIST;
		
	else if( CAN_WEAR( obj, ITEM_EQUIP_SHOULDER ) )
		return EQUIP_SHOULDER;
		
	else if( CAN_WEAR( obj, ITEM_EQUIP_CLOAK ) )
		return EQUIP_CLOAK;
		
	else if( CAN_WEAR( obj, ITEM_EQUIP_BACK ) )
		return EQUIP_BACK;
		
	else if( CAN_WEAR( obj, ITEM_EQUIP_PIN ) )
		return EQUIP_PIN;
		
	/* else if( CAN_WEAR( obj, ITEM_EQUIP_RIGHTHAND ) )
		return EQUIP_RIGHTHAND;
		
	else if( CAN_WEAR( obj, ITEM_EQUIP_LEFTHAND ) )
		return EQUIP_LEFTHAND; */
		
	else
		return EQUIP_NONE;	
}


void new_wear_obj( CHAR_DATA *ch, OBJ_DATA *obj )
{
	int loc = new_get_wear_loc( obj );
	
	if ( loc == EQUIP_NONE || obj->item_type == ITEM_WEAPON )
	{
		send_to_char( "You can't wear that!\r\n", ch );
		return;
	}
	
	if( get_num_equipped( ch, loc ) + 1 > new_get_max_worn( loc ) )
	{
		send_to_char( "You are already wearing as much as you can in that location.\r\n", ch );
		printf_to_char( ch, "Loc: %d Equipped: %d Max: %d\r\n", loc, 
		get_num_equipped( ch, loc ),
		new_get_max_worn( loc ) );
		return;
	}
	else
	{
		if( obj->item_type != ITEM_SHIELD )
		{
		act( "$n puts on $p.", ch, obj, NULL, TO_ROOM );
		act( "You put on $p.",  ch, obj, NULL, TO_CHAR );
		}
		else
		{
		act( "$n slings $p over $s shoulder.", ch, obj, NULL, TO_ROOM );
		act( "You sling $p over your shoulder.",  ch, obj, NULL, TO_CHAR );	
		}
		
		equip_char( ch, obj, loc );
		//fixme roundtime for armor
		return;
	}
}



void do_wear( CHAR_DATA *ch, char *argument )
{
    char arg[MAX_INPUT_LENGTH];
    OBJ_DATA *obj;

    one_argument( argument, arg );

    if ( arg[0] == '\0' )
    {
	send_to_char( "Wear, wield, or hold what?\r\n", ch );
	return;
    }

	if ( ( obj = get_obj_carry( ch, arg ) ) == NULL )
	{
	    send_to_char( "You do not have that item.\r\n", ch );
	    return;
	}

	new_wear_obj( ch, obj );

    return;
}


void do_remove( CHAR_DATA *ch, char *argument )
{
    char arg[MAX_INPUT_LENGTH];
    OBJ_DATA *obj;

    one_argument( argument, arg );

    if ( arg[0] == '\0' )
    {
	send_to_char( "Remove what?\r\n", ch );
	return;
    }

    if ( ( obj = get_obj_wear( ch, arg ) ) == NULL  )
    {
	send_to_char( "You are not wearing anything like that.\r\n", ch );
	return;
    }
	
	if( obj->wear_loc == EQUIP_NONE || obj->wear_loc == EQUIP_RIGHTHAND  || obj->wear_loc == EQUIP_LEFTHAND )
    {
	send_to_char( "You are not wearing that, though.\r\n", ch );
	return;
    }	

	if( get_eq_char( ch, EQUIP_RIGHTHAND ) != NULL &&  get_eq_char( ch, EQUIP_LEFTHAND ) != NULL )
    {
	send_to_char( "You need a free hand.\r\n", ch );
	return;
    }
    
	if ( IS_SET(obj->extra_flags, ITEM_NOREMOVE) )
    {
	act( "As you try to remove your $o, it glows with a dark aura and repels your touch!", ch, obj, NULL, TO_CHAR );
	return;
    }

    remove_obj( ch, obj );
    
    if( obj->item_type != ITEM_SHIELD )
	obj_to_free_hand( obj, ch );
    else
    	obj_to_off_hand( obj, ch );
	
    return;
}



void do_sacrifice( CHAR_DATA *ch, char *argument )
{
    return;	
}


int get_gsn( int number )
{
    int sn;

    for ( sn = 0; sn < MAX_SPELL; sn++ )
    {
		if( spell_table[sn].number == number ) 
		return sn;
	}

    return -1;
}




//fixme scrolls by spell #, also need skill check for runes
void do_recite( CHAR_DATA *ch, char *argument )
{
    char arg1[MAX_INPUT_LENGTH];
    char arg2[MAX_INPUT_LENGTH];
    CHAR_DATA *victim;
    OBJ_DATA *scroll;
    OBJ_DATA *obj;

    argument = one_argument( argument, arg1 );
    argument = one_argument( argument, arg2 );

    if ( ( scroll = get_obj_carry( ch, arg1 ) ) == NULL )
    {
	send_to_char( "You do not have that scroll.\r\n", ch );
	return;
    }

    if ( scroll->item_type != ITEM_SCROLL )
    {
	send_to_char( "You can recite only scrolls.\r\n", ch );
	return;
    }

    obj = NULL;
    if ( arg2[0] == '\0' )
    {
	victim = ch;
    }
    else
    {
	if ( ( victim = get_char_room ( ch, arg2 ) ) == NULL
	&&   ( obj    = get_obj_here  ( ch, arg2 ) ) == NULL )
	{
	    send_to_char( "You can't find it.\r\n", ch );
	    return;
	}
    }
    act( "$n recites $p.", ch, scroll, NULL, TO_ROOM );
    act( "You recite $p.", ch, scroll, NULL, TO_CHAR );

    obj_cast_spell( scroll->value[0], scroll->value[0], ch, victim, obj );
    obj_cast_spell( scroll->value[1], scroll->value[0], ch, victim, obj );
    obj_cast_spell( scroll->value[2], scroll->value[0], ch, victim, obj );
    obj_cast_spell( scroll->value[3], scroll->value[0], ch, victim, obj );

    extract_obj( scroll );
    return;
}

void do_rub( CHAR_DATA *ch, char *argument )
{
    CHAR_DATA *victim;
    OBJ_DATA *item;
    OBJ_DATA *obj;
	int lvl, gsn;

    if ( ( item = get_obj_here( ch, argument ) ) == NULL )
    {
	printf_to_char( ch, "But where is %s to rub?\r\n", argument );
	return;
    }

    if ( item->item_type != ITEM_RUB )
    {
	act( "$n rubs $p.", ch, item, NULL, TO_ROOM );
    act( "You rub $p but nothing happens.", ch, item, NULL, TO_CHAR );
	return;
    }

    obj = NULL;
	victim = ch;
	
	if( !IS_NPC( ch ) )
	lvl = ch->pcdata->learned[gsn_magic_item_use];
	else
	lvl = ch->level;
	
	gsn = get_gsn(item->value[0]);
   
    act( "$n rubs $p.", ch, item, NULL, TO_ROOM );
    act( "You rub $p.", ch, item, NULL, TO_CHAR );
	
	if( gsn == -1 )
	{
	send_to_char( "Nothing happens.\r\n", ch );
	return;
	}
	else
    obj_cast_spell( gsn, lvl, ch, victim, obj );
	
	//fixme items that persist? do_lick (metallic)
	if ( --item->value[1] <= 0 )
    {
	act( "$p crumbles into dust.", ch, item, NULL, TO_ROOM );
	act( "$p crumbles into dust.", ch, item, NULL, TO_CHAR );
	extract_obj( item );
    }
	
	add_roundtime( ch, 3 );
	show_roundtime( ch, 3 );

    return;
}

void do_wave( CHAR_DATA *ch, char *argument )
{
    char arg1[MAX_INPUT_LENGTH];
    char arg2[MAX_INPUT_LENGTH];
    CHAR_DATA *victim;
    OBJ_DATA *scroll;
    OBJ_DATA *obj;
	int lvl, gsn;
	
	if( argument == NULL )
	{
    act( "You wave.", ch, NULL, NULL, TO_CHAR );
    act( "$n waves.", ch, NULL, NULL, TO_ROOM );
    return;
    }	
	
	victim =  get_char_room( ch, argument );

    if( victim != NULL )
    {
	
	if (ch == victim )
	{
	send_to_char( "You flap your hands around awkardly.\r\n", ch );
	act( "$n flaps $s hands around.", ch, NULL, NULL, TO_ROOM );
	return;
    }
		
    act( "You wave to $N.", ch, NULL, victim, TO_CHAR );
    act( "$n waves to you.", ch, NULL, victim, TO_ROOM );
    return;
    }

    argument = one_argument( argument, arg1 );
    argument = one_argument( argument, arg2 );
	
	if (arg1[0] == '\0' )
	{
	send_to_char( "Wave what?  Or at what?\r\n", ch );
	return;
    }

    if ( ( scroll = get_obj_here( ch, arg1 ) ) == NULL )
    {
	send_to_char( "Nothing like that here.\r\n", ch );
	return;
    }

    if ( scroll->item_type == ITEM_WAND )
    {
	
	
    obj = NULL;
	
	if( !IS_NPC( ch ) )
	lvl = ch->pcdata->learned[gsn_magic_item_use];
	else
	lvl = ch->level;
	
	gsn = get_gsn(scroll->value[0]);
	
    if ( arg2[0] == '\0' )
    {
	victim = ch;
    }
    else
    {
	if ( ( victim = get_char_room ( ch, arg2 ) ) == NULL
	&&   ( obj    = get_obj_here  ( ch, arg2 ) ) == NULL )
	{
	    send_to_char( "You can't find it.\r\n", ch );
	    return;
	}
    }

//fixme at himself...
	if( victim == ch )
	{
	    act( "$n waves $p at $mself.", ch, scroll, NULL, TO_ROOM );
	    act( "You wave $p at yourself.", ch, scroll, NULL, TO_CHAR );
	}
	else if ( victim != NULL )
	{
	    act( "$n waves $p at $N.", ch, scroll, victim, TO_ROOM );
	    act( "You wave $p at $N.", ch, scroll, victim, TO_CHAR );
	}
	else if ( obj != NULL )
	{
	    act( "$n waves $p at $P.", ch, scroll, obj, TO_ROOM );
	    act( "You wave $p at $P.", ch, scroll, obj, TO_CHAR );
	}

	if( gsn == -1 )
	{
	send_to_char( "Nothing happens.\r\n", ch );
	return;
	}
	else
    obj_cast_spell( gsn, lvl, ch, victim, obj );

    if ( --scroll->value[1] <= 0 )
    {
	act( "$p crumbles into dust.", ch, scroll, NULL, TO_ROOM );
	act( "$p crumbles into dust.", ch, scroll, NULL, TO_CHAR );
	extract_obj( scroll );
    }
	
	}
	
    return;
}





/*
 * Shopping commands.
 */
CHAR_DATA *find_keeper( CHAR_DATA *ch )
{
    CHAR_DATA *keeper;
    SHOP_DATA *pShop;

    pShop = NULL;
    for ( keeper = ch->in_room->people; keeper; keeper = keeper->next_in_room )
    {
	if ( IS_NPC(keeper) && (pShop = keeper->pIndexData->pShop) != NULL )
	    break;
    }

    if ( pShop == NULL )
    {
	send_to_char( "You can't do that here.\r\n", ch );
	return NULL;
    }

    /*
     * Shop hours.
     */
	 
	 /*
    if ( time_info.hour < pShop->open_hour )
    {
	do_say( keeper, "Sorry, come back later." );
	return NULL;
    }

    if ( time_info.hour > pShop->close_hour )
    {
	do_say( keeper, "Sorry, come back tomorrow." );
	return NULL;
    }
	*/

    /*
     * Invisible or hidden people.
     */
    if ( !can_see( keeper, ch ) )
    {
	do_say( keeper, "Is someone there?" );
	return NULL;
    }

    return keeper;
}



int get_cost( CHAR_DATA *keeper, OBJ_DATA *obj, bool fBuy )
{
    SHOP_DATA *pShop;
    int cost;

    if ( obj == NULL || ( pShop = keeper->pIndexData->pShop ) == NULL )
	return 0;

    if ( fBuy )
    {
	cost = obj->cost * pShop->profit_buy  / 100;
    }
    else
    {
	OBJ_DATA *obj2;
	int itype;

	cost = 0;
	for ( itype = 0; itype < MAX_TRADE; itype++ )
	{
	    if ( obj->item_type == pShop->buy_type[itype] )
	    {
		cost = obj->cost * pShop->profit_sell / 100;
		break;
	    }
	}

	for ( obj2 = keeper->carrying; obj2; obj2 = obj2->next_content )
	{
	    if ( obj->pIndexData == obj2->pIndexData )
	    {
		cost = 0;
		break;
	    }
	}
    }

    if ( obj->item_type == ITEM_STAFF || obj->item_type == ITEM_WAND )
	cost = cost * obj->value[2] / obj->value[1];

    return cost;
}

void do_order2( CHAR_DATA *ch, char *argument )
{
	char arg[MAX_INPUT_LENGTH];
	int i;
	int anum;

    argument = one_argument( argument, arg );
	
	if ( arg[0] == '\0' )
    {
		if( ch->in_room->vnum == ROOM_VNUM_WEAPON_SHOP || 
			ch->in_room->vnum == ROOM_VNUM_WEAPON_SHOP2		)
		{
			for( i = 1; i < MAX_WEAPON-4; i++ )
			{
				printf_to_char( ch, "%2d) %s\r\n", i, weapon_table[i].name );
			}
		}
	
		else if( ch->in_room->vnum == ROOM_VNUM_ARMOR_SHOP ||
			ch->in_room->vnum == ROOM_VNUM_ARMOR_SHOP2 )
		{
			for( i = 1; i < MAX_ARMOR; i++ )
			{
				printf_to_char( ch, "%2d) %s\r\n", i, armor_table[i].name );
			}
			
			while( i < (MAX_SHIELD+MAX_ARMOR) )
			{
				printf_to_char( ch, "%2d) %s\r\n", i, shield_table[i-MAX_ARMOR].name );
				i++;
			}
			
			
		}
		
	/*	else if( ch->in_room->vnum == ROOM_VNUM_YS_BAR1 || 
		ch->in_room->vnum == ROOM_VNUM_YS_BAR2 || 
		ch->in_room->vnum == ROOM_VNUM_YS_BAR3 || 
		ch->in_room->vnum == ROOM_VNUM_YS_BAR4	||
		ch->in_room->vnum == ROOM_VNUM_SCUNTHORPE_TAVERN		)
		{
					//fixme booze
			printf_to_char(ch, "You place an order for some food and drink.\r\n" );
			printf_to_char(ch, "The staff regretfully inform you there is none at the moment.\r\n" );
			return;
		} */
		else
		{
			printf_to_char( ch, "You can't order anything here.\r\n" );
			return;
		}
		
	return;
    }
	
	if ( is_number( arg ) )
	{
	    anum = atoi( arg );
		
		i = anum;
	}
	else
	{
		printf_to_char( ch, "Order what number?\r\n" );
		return;
	}
	
	if( ch->in_room->vnum == ROOM_VNUM_WEAPON_SHOP || 
		ch->in_room->vnum == ROOM_VNUM_WEAPON_SHOP2	)
	{
		if( i < 1 || i >= MAX_WEAPON-4 )
		{
		printf_to_char( ch, "No such number to order: %d.\r\n", i );
		return;
		}

		ch->order = i;
		ch->ordercost = weapon_table[i].cost;
		
		printf_to_char(ch, "That'll be %d for %s %s.\r\n", weapon_table[i].cost, select_a_an( weapon_table[i].name ), weapon_table[i].name );
		printf_to_char(ch, "If you accept this price, BUY now.\r\n" );
	}
	else if( ch->in_room->vnum == ROOM_VNUM_ARMOR_SHOP || 
		ch->in_room->vnum == ROOM_VNUM_ARMOR_SHOP2	)
	{
		if( i < 1 || i >= MAX_ARMOR+MAX_SHIELD )
		{
		printf_to_char( ch, "No such number to order: %d.\r\n", i );
		return;
		}
		
		if( i >= MAX_ARMOR )
		{
		short j = i-MAX_ARMOR;
		ch->order = i;
		ch->ordercost = shield_table[j].cost;
		
		printf_to_char(ch, "ORDER %d - SHIELD %d --- %s -- costs %d.\r\n", ch->order, j, shield_table[j].name, ch->ordercost);
		
		printf_to_char(ch, "That'll be %d for %s %s.\r\n", shield_table[j].cost, select_a_an( shield_table[j].name ), shield_table[j].name );
		printf_to_char(ch, "If you accept this price, BUY now.\r\n" );
		}
		else
		{
		ch->order = i;
		ch->ordercost = armor_table[i].cost;
		
		printf_to_char(ch, "That'll be %d for %s %s.\r\n", armor_table[i].cost, i < 21 ? "some" : select_a_an( armor_table[i].name ), armor_table[i].name );
		printf_to_char(ch, "If you accept this price, BUY now.\r\n" );
		}
	}
	else if( ch->in_room->vnum == ROOM_VNUM_YS_BAR1 || 
		ch->in_room->vnum == ROOM_VNUM_YS_BAR2 || 
		ch->in_room->vnum == ROOM_VNUM_YS_BAR3 || 
		ch->in_room->vnum == ROOM_VNUM_YS_BAR4	||
		ch->in_room->vnum == ROOM_VNUM_SCUNTHORPE_TAVERN		)
	{
		//fixme booze
		printf_to_char(ch, "The staff regretfully informs you that they are out of food and drink at the moment.\r\n" );
		return;
	}
	else
		return;
			
}

OBJ_DATA *create_armor( int number  )
{
	char buf[MSL];
	OBJ_DATA *armor = create_object( get_obj_index( OBJ_VNUM_ARMOR ), 0 );
	int num = number;
	
	strcpy( buf, "" );
	
	sprintf(buf, "%s", armor_table[num].name );	
	armor->name = str_dup(buf);
	
	strcpy( buf, "" );
	
	if( num <= 20 )
	{
	sprintf(buf, "some %s",  armor_table[num].name );	
	armor->short_descr = str_dup( buf );
	}
	else
	{
	sprintf(buf, "%s %s", select_a_an( armor_table[num].name ), armor_table[num].name );	
	armor->short_descr = str_dup( buf );
	}
	
	armor->wear_flags = armor_table[num].worn;
	
	armor->value[0] = armor_table[num].v0;
	armor->value[1] = armor_table[num].v1;
	armor->value[2] = armor_table[num].v2;	 
	armor->value[3] = armor_table[num].v3;
				
	armor->weight = armor_table[num].weight;
	armor->cost = armor_table[num].cost;
	
	return armor;
}

OBJ_DATA *create_shield( int number  )
{
	char buf[MSL];
	OBJ_DATA *shield = create_object( get_obj_index( OBJ_VNUM_SHIELD ), 0 );
	int num = number;
	
	strcpy( buf, "" );
	
	sprintf(buf, "%s", shield_table[num].name );	
	shield->name = str_dup(buf);
	
	strcpy( buf, "" );
	
	sprintf(buf, "%s %s", select_a_an( shield_table[num].name ), shield_table[num].name );	
	shield->short_descr = str_dup( buf );
	
	shield->st = shield_table[num].st;
	shield->du = shield_table[num].du; 
	
	shield->value[0] = num;
	shield->value[1] = shield_table[num].size;
	shield->value[2] = 0; //0x
	shield->value[3] = 0;
	
	shield->weight = shield_table[num].weight;
	shield->cost = shield_table[num].cost;
	
	return shield;
}


OBJ_DATA *create_weapon( int number  )
{
	char buf[MSL];
	OBJ_DATA *weapon = create_object( get_obj_index( OBJ_VNUM_WEAPON ), 0 );
	int num = number;
		
	strcpy( buf, "" );
	
	sprintf(buf, "%s", weapon_table[num].name );	
	weapon->name = str_dup(buf);
	
	strcpy( buf, "" );
	
	sprintf(buf, "%s %s", select_a_an(  weapon_table[num].name ),  weapon_table[num].name );	
	weapon->short_descr = str_dup( buf );
	
	weapon->value[0] = num;
	weapon->value[1] = 0;
	weapon->value[2] = 0;
	weapon->value[3] = 0;
				
	weapon->weight = weapon_table[num].weight;
	weapon->cost = weapon_table[num].cost;

	return weapon;
}

OBJ_DATA *create_random_magic_item( int number  )
{
        char buf[MSL];
        OBJ_DATA *magic_item = create_object( get_obj_index( OBJ_VNUM_WAND ), 0 );
        int num = number;

        strcpy( buf, "" );

        sprintf(buf, "%s", magic_item_table[num].name );    
        magic_item->name = str_dup(buf);

        strcpy( buf, "" );
        
        sprintf(buf, "%s %s", select_a_an(  magic_item_table[num].name ),  magic_item_table[num].name );        
        magic_item->short_descr = str_dup( buf );

	magic_item->item_type = magic_item_table[num].item_type;

        magic_item->value[0] = magic_item_table[num].spell;
        magic_item->value[1] = magic_item_table[num].charges;
        magic_item->value[2] = 0;
        magic_item->value[3] = 0;

        magic_item->weight = magic_item_table[num].weight;
        magic_item->cost = magic_item_table[num].cost;

        return magic_item;
}

void do_buy( CHAR_DATA *ch, char *argument )
{
    char arg1[MAX_INPUT_LENGTH];
    char arg2[MAX_INPUT_LENGTH];
    OBJ_DATA *obj;
    OBJ_DATA *container;
	bool forSale = FALSE;
	
    argument = one_argument( argument, arg1 );
    argument = one_argument( argument, arg2 );

    /* Get type. */
    if ( arg1[0] == '\0' )
    {
	do_buy2( ch, "" );
	return;
    }

	// 1 arg.
    if ( arg2[0] == '\0' )
    {
	    /* 'buy obj' */			
		obj = get_obj_list( ch, arg1, ch->in_room->contents );
		
		if( obj == NULL )
		{
		obj = get_obj_in_room_containers( ch, arg1 ); 
		
		if( obj != NULL )
		{
			container = get_obj_in_room_containers2( ch, arg1 ); 
			
			if ( container->item_type == ITEM_MERCHANT_TABLE )
				forSale = TRUE;
		}
		}
		
	    if ( obj == NULL )
	    {
		send_to_char( "Buy what?\r\n", ch );
		return;
	    }
		
		if( forSale == FALSE )
		{
	    send_to_char( "But it is not for sale.\r\n", ch );
	    return;
		}
		
		// the item on the table remains there, for sale.  A copy is bought by the character.
		{
		OBJ_DATA *dupe;
		
		dupe = create_object( obj->pIndexData, 0 );
		
		printf_to_char( ch, "You hand over %d silver coins and receive %s in return.\r\n", dupe->cost, dupe->short_descr );
		act( "$n purchases $p on $P.", ch, obj, container, TO_ROOM );
		obj_to_char( dupe, ch );
		obj_to_free_hand( dupe, ch );
		ch->gold -= dupe->cost;
		}
    }
    else
    {
	/* 'buy ... container' */

	if ( ( container = get_obj_here( ch, arg2 ) ) == NULL )
	{
		send_to_char("Buy from what?\r\n", ch);
	    return;
	}

	if( container->item_type != ITEM_MERCHANT_TABLE )
	{
	    send_to_char( "But nothing from it is for sale.\r\n", ch );
	    return;
	}

	    //'buy obj container'
	    obj = get_obj_list( ch, arg1, container->contains );
	    if ( obj == NULL )
	    {
		send_to_char("Buy what from what?\r\n", ch);
		return;
	    }
	    // buy it...
		
		if( ch->gold < obj->cost )
		{
		send_to_char("You do not have enough silver coins.\r\n", ch);
		return;
		}
		
		if( get_eq_char( ch, EQUIP_RIGHTHAND ) != NULL &&  get_eq_char( ch, EQUIP_LEFTHAND ) != NULL )
		{
		send_to_char( "You need a free hand.\r\n", ch );
		return;
		}
		
		// the item on the table remains there, for sale.  A copy is bought by the character.
		{
		OBJ_DATA *dupe;
		
		dupe = create_object( obj->pIndexData, 0 );
		
		printf_to_char( ch, "You hand over %d silver coins and receive %s in return.\r\n", dupe->cost, dupe->short_descr );
		act( "$n purchases $p on $P.", ch, obj, container, TO_ROOM );
		obj_to_char( dupe, ch );
		obj_to_free_hand( dupe, ch );
		ch->gold -= dupe->cost;
		}
		
		return;
    }

    return;
}


void do_buy2( CHAR_DATA *ch, char *argument )
{
    char arg[MAX_INPUT_LENGTH];

    argument = one_argument( argument, arg );

	if ( arg[0] != '\0' || ch->order == 0 || ch->ordercost == 0 )
    {
	send_to_char( "Buy what?\r\n", ch );
	return;
    }

	if( ch->gold < ch->ordercost)
	{
	send_to_char( "Only buy what you can pay for.\r\n", ch );
	return;
    }
	
	if( ch->in_room->vnum == ROOM_VNUM_ARMOR_SHOP || ch->in_room->vnum == ROOM_VNUM_ARMOR_SHOP2  )
	{
		if( ch->order < MAX_ARMOR )
		{
		printf_to_char(ch, "You hand over %d silver coins for %s %s.\r\n", armor_table[ch->order].cost, ch->order < 21 ? "some" : select_a_an( armor_table[ch->order].name ), armor_table[ch->order].name );
		obj_to_room( create_armor( ch->order ), ch->in_room );
		}
		else
		{
		ch->order -= MAX_ARMOR;
		
		if( ch->order < MAX_SHIELD && ch->order >= 0 )
		{		
		//printf_to_char(ch, "SHIELD %d --- %s -- costs %d.\r\n", ch->order, shield_table[ch->order].name, ch->ordercost);		
		printf_to_char(ch, "You hand over %d silver coins for %s %s.\r\n", shield_table[ch->order].cost, select_a_an( shield_table[ch->order].name ), shield_table[ch->order].name );
		obj_to_room( create_shield( ch->order ), ch->in_room );
		}
		else
		printf_to_char(ch, "Incorrect order: %d.\r\n", ch->order );	
		}	
		
		ch->gold -= ch->ordercost;
		
		ch->order = 0;
		ch->ordercost = 0;
		return;
	}
	else if( ch->in_room->vnum == ROOM_VNUM_WEAPON_SHOP || ch->in_room->vnum == ROOM_VNUM_WEAPON_SHOP2)
	{
		printf_to_char(ch, "You hand over %d silver coins for %s %s.\r\n", weapon_table[ch->order].cost, select_a_an( weapon_table[ch->order].name ), weapon_table[ch->order].name );
		obj_to_room( create_weapon( ch->order ), ch->in_room );
		
		ch->gold -= ch->ordercost;
		
		ch->order = 0;
		ch->ordercost = 0;
		return;
	}
	else
	{
	send_to_char( "You can't buy anything here.\r\n", ch );
	return;
    }
}


void do_sell2( CHAR_DATA *ch, char *argument )
{
    char arg[MAX_INPUT_LENGTH];
    OBJ_DATA *obj;
    int cost;
	bool proper = FALSE;
	bool here = FALSE;

    one_argument( argument, arg );

    if ( arg[0] == '\0' )
    {
	send_to_char( "Sell what?\r\n", ch );
	return;
    }

    if ( ( obj = get_obj_carry( ch, arg ) ) == NULL )
    {
	send_to_char( "Sell what?\r\n", ch );
	return;
    }
	
    if ( !can_drop_obj( ch, obj ) )
    {
	send_to_char( "You can't let go of it.\r\n", ch );
	return;
    }
	
	if( IS_SET( obj->extra_flags, ITEM_GENERATED ) )
	{
	send_to_char( "You cannot sell that.\r\n", ch );
	return;
    }
	
	if( obj->item_type == ITEM_FOOD || obj->item_type == ITEM_DRINK_CON )
	{
	send_to_char( "It is too perishable to sell.\r\n", ch );
	return;
    }	
	
	if( obj->item_type == ITEM_ARMOR || obj->item_type == ITEM_SHIELD )
	{
		switch( ch->in_room->vnum )
		{
			case ROOM_VNUM_ARMOR_SHOP: case ROOM_VNUM_ARMOR_SHOP2: 
			proper = TRUE; here = TRUE; 
			break;
			
			case ROOM_VNUM_WEAPON_SHOP: 
			case ROOM_VNUM_WEAPON_SHOP2: 
			here = TRUE; break;
			
			case ROOM_VNUM_PAWNSHOP: 
			here = TRUE; break;
		}	
	}
	else if( obj->item_type == ITEM_WEAPON )
	{
		switch( ch->in_room->vnum )
		{
			case ROOM_VNUM_WEAPON_SHOP: case ROOM_VNUM_WEAPON_SHOP2: 
			proper = TRUE; here = TRUE; 
			break;
			
			case ROOM_VNUM_ARMOR_SHOP: 
			case ROOM_VNUM_ARMOR_SHOP2: 
			here = TRUE; break;
			
			case ROOM_VNUM_PAWNSHOP: 
			here = TRUE; break;
		}	
	}
	else if( obj->item_type == ITEM_POTION || obj->item_type == ITEM_SCROLL || obj->item_type == ITEM_RUB || obj->item_type == ITEM_WAND )
	{
		switch( ch->in_room->vnum )
		{
			case ROOM_VNUM_MAGIC_SHOP: case ROOM_VNUM_YS_MAGIC_SHOP: 
			proper = TRUE; here = TRUE; 
			break;
			
			case ROOM_VNUM_PAWNSHOP: 
			here = TRUE; break;
		}	
	}
	else if( obj->item_type == ITEM_GEM )
	{
		switch( ch->in_room->vnum )
		{
			case ROOM_VNUM_GEMSHOP: case ROOM_VNUM_YS_GEMSHOP: proper = TRUE; here = TRUE; 
			break;
			case ROOM_VNUM_PAWNSHOP: here = TRUE;
			break;
		}
	}
	else if( obj->item_type == ITEM_SKIN )
	{
		switch( ch->in_room->vnum )
		{
			case ROOM_VNUM_FURRIER: case ROOM_VNUM_YS_FURRIER: proper = TRUE; here = TRUE; 
			break;
			case ROOM_VNUM_PAWNSHOP: here = TRUE;
			break;
		}
	}
	else
	{
		if( ch->in_room->vnum == ROOM_VNUM_PAWNSHOP )
			here = TRUE;
	}
	
	cost = (obj->cost / 4);
	
	if (proper)
	cost = cost * 2;
	//fixme trading and charm bonus
	
	if ( !here )
    {
	send_to_char( "You cannot sell that here.\r\n", ch );
	return;
    }

    if ( cost <= 0 || obj->item_type == ITEM_TRASH || IS_SET( obj->extra_flags, ITEM_GENERATED) )
    {
	send_to_char( "The shopkeeper refuses to buy it.\r\n", ch );
	return;
    }

	printf_to_char( ch, "You sell %s and receive %d silver coins in return.\r\n", obj->short_descr, cost );
	
    ch->gold     += cost;	
	obj_from_char( obj );
	extract_obj( obj );

    return;
}



void do_value( CHAR_DATA *ch, char *argument )
{
    char arg[MAX_INPUT_LENGTH];
    OBJ_DATA *obj;
    int cost;
	bool proper = FALSE;
	bool here = FALSE;

    one_argument( argument, arg );

    if ( arg[0] == '\0' )
    {
	send_to_char( "Appraise what?\r\n", ch );
	return;
    }

    if ( ( obj = get_obj_carry( ch, arg ) ) == NULL )
    {
	send_to_char( "Appraise what?\r\n", ch );
	return;
    }
	
    if ( !can_drop_obj( ch, obj ) )
    {
	send_to_char( "You can't let go of it.\r\n", ch );
	return;
    }
	
	if( IS_SET( obj->extra_flags, ITEM_GENERATED ) )
	{
	send_to_char( "You cannot appraise that.\r\n", ch );
	return;
    }
	
	if( obj->item_type == ITEM_FOOD || obj->item_type == ITEM_DRINK_CON )
	{
	send_to_char( "It is too perishable to appraise.\r\n", ch );
	return;
    }
	
	if( obj->item_type == ITEM_ARMOR || obj->item_type == ITEM_SHIELD )
	{
		switch( ch->in_room->vnum )
		{
			case ROOM_VNUM_ARMOR_SHOP: case ROOM_VNUM_ARMOR_SHOP2: 
			proper = TRUE; here = TRUE; 
			break;
			
			case ROOM_VNUM_WEAPON_SHOP: 
			case ROOM_VNUM_WEAPON_SHOP2: 
			here = TRUE; break;
			
			case ROOM_VNUM_PAWNSHOP: 
			here = TRUE; break;
		}	
	}
	else if( obj->item_type == ITEM_WEAPON )
	{
		switch( ch->in_room->vnum )
		{
			case ROOM_VNUM_WEAPON_SHOP: case ROOM_VNUM_WEAPON_SHOP2: 
			proper = TRUE; here = TRUE; 
			break;
			
			case ROOM_VNUM_ARMOR_SHOP: 
			case ROOM_VNUM_ARMOR_SHOP2: 
			here = TRUE; break;
			
			case ROOM_VNUM_PAWNSHOP: 
			here = TRUE; break;
		}	
	}
	else if( obj->item_type == ITEM_POTION || obj->item_type == ITEM_SCROLL || obj->item_type == ITEM_RUB || obj->item_type == ITEM_WAND )
	{
		switch( ch->in_room->vnum )
		{
			case ROOM_VNUM_MAGIC_SHOP: case ROOM_VNUM_YS_MAGIC_SHOP: 
			proper = TRUE; here = TRUE; 
			break;
			
			case ROOM_VNUM_PAWNSHOP: 
			here = TRUE; break;
		}	
	}
	else if( obj->item_type == ITEM_GEM )
	{
		switch( ch->in_room->vnum )
		{
			case ROOM_VNUM_GEMSHOP: case ROOM_VNUM_YS_GEMSHOP: proper = TRUE; here = TRUE; 
			break;
			case ROOM_VNUM_PAWNSHOP: here = TRUE;
			break;
		}
	}
	else if( obj->item_type == ITEM_SKIN )
	{
		switch( ch->in_room->vnum )
		{
			case ROOM_VNUM_FURRIER: case ROOM_VNUM_YS_FURRIER: proper = TRUE; here = TRUE; 
			break;
			case ROOM_VNUM_PAWNSHOP: here = TRUE;
			break;
		}
	}
	else
	{
		if( ch->in_room->vnum == ROOM_VNUM_PAWNSHOP )
			here = TRUE;
	}
	
	cost = (obj->cost / 4);
	
	if (proper)
	cost = cost * 2;
	//fixme trading and charm bonus
	
	if ( !here )
    {
	send_to_char( "You cannot appraise that here.\r\n", ch );
	return;
    }

    if ( cost <= 0 || obj->item_type == ITEM_TRASH || IS_SET( obj->extra_flags, ITEM_GENERATED) )
    {
	send_to_char( "The shopkeeper refuses to appraise it.\r\n", ch );
	return;
    }

	printf_to_char( ch, "You would get %d coins for your %s.\r\n", cost, smash_article(obj->short_descr) );

    return;
}

void do_swap( CHAR_DATA *ch, char *argument )
{
	OBJ_DATA *obj1 = get_eq_char( ch, EQUIP_RIGHTHAND );
	OBJ_DATA *obj2 = get_eq_char( ch, EQUIP_LEFTHAND );
	
	if( obj1 != NULL && obj2 != NULL )
	{
		printf_to_char( ch, "You swap %s to your right hand and %s to your left hand.\r\n",
			obj2->short_descr, obj1->short_descr ) ;
		
		obj2->wear_loc = EQUIP_RIGHTHAND;
		obj1->wear_loc = EQUIP_LEFTHAND;
		return;
	}
	else if( obj1 == NULL && obj2 != NULL )
	{
		printf_to_char( ch, "You swap %s to your right hand.\r\n",
			obj2->short_descr ) ;
		
		obj2->wear_loc = EQUIP_RIGHTHAND;
		return;
	}
	else if( obj1 != NULL && obj2 == NULL )
	{
		printf_to_char( ch, "You swap %s to your left hand.\r\n",
			obj1->short_descr ) ;
		
		obj1->wear_loc = EQUIP_LEFTHAND;
		return;
	}
	else
	{
		send_to_char( "You don't have anything to swap!\r\n", ch );
		return;
	}
}

void do_peer (CHAR_DATA *ch, char *argument)
{
	OBJ_DATA *obj;
	CHAR_DATA *victim;
	EXIT_DATA *pexit;
	ROOM_INDEX_DATA *to_room;
	ROOM_INDEX_DATA *orig;
	int door;
	extern char * const dir_name[];

    if ( ch->desc == NULL )
		return;

	if ( argument[0] == '\0'  )
    {
	send_to_char( "Peer at whom or what?\r\n", ch );
	return;
    }
	
    if ( ( victim = get_char_room( ch, argument ) ) != NULL )
    {
	
	if( victim != ch )
	{
	act( "$n peers quizzically at $N.", ch, NULL, victim, TO_NOTVICT );
	act( "$n peers quizzically at you.", ch, NULL, victim, TO_VICT );
	
	printf_to_char( ch, "You peer quizzically at %s.\r\n", HIM_HER(victim) );
	return;
	}
	else
	{
	act( "$n crosses $s eyes.", ch, NULL, NULL, TO_ROOM );
	printf_to_char( ch, "You cross your eyes.\r\n"  );
	return;
	}
	
	return;
    }

    for ( obj = ch->carrying; obj != NULL; obj = obj->next_content )
    {
	if ( can_see_obj( ch, obj ) && is_name( argument, obj->name ) )
	{
		act( "You peer quizzically at your $o.", ch, obj, NULL, TO_CHAR );
		act( "$n peers quizzically at $s $o.", ch, obj, NULL, TO_ROOM );	
		return;
	}
    }

    for ( obj = ch->in_room->contents; obj != NULL; obj = obj->next_content )
    {
	if ( can_see_obj( ch, obj ) && is_name( argument, obj->name ) )
	{
		if( obj->item_type == ITEM_DOOR || obj->item_type == ITEM_PORTAL || obj->item_type == ITEM_BUILDING || obj->item_type == ITEM_CLIMB || obj->item_type == ITEM_SWIM )
		{
			printf_to_char(ch, "You peer through %s and see...\r\n", obj->short_descr );
			to_room = get_room_index( obj->value[0] );
			
			if( to_room != NULL )
			{
			orig = ch->in_room;

			char_from_room( ch );
			char_to_room( ch, to_room );
			
			do_look(ch, "");
			
			char_from_room( ch );
			char_to_room( ch, orig );
			}
			
			return;
		}
		
		else
		{
		act( "You peer quizzically at $p.", ch, obj, NULL, TO_CHAR );
		act( "$n peers quizzically at $p.", ch, obj, NULL, TO_ROOM );	
		return;
		}			
		
	    return;
	}
    }
	
	//fixme
	door = get_direction(argument);
	
	if ( door == -1 || ( pexit = ch->in_room->exit[door] ) == NULL )
    {
	send_to_char( "You cannot peer in that direction.\n\r", ch );
	return;
    }
	else
	{

    if ( pexit->description != NULL && pexit->description[0] != '\0' )
	send_to_char( pexit->description, ch );
	else
	printf_to_char(ch, "You peer %s and see...\r\n", dir_name[door] );
	
	if( pexit->to_room != NULL )
	{
	orig = ch->in_room;
	to_room = pexit->to_room;
	
	char_from_room( ch );
	char_to_room( ch, to_room );
	
	do_look(ch, "");
	
	char_from_room( ch );
	char_to_room( ch, orig );
	}
	
	return;
	}
	
	printf_to_char(ch, "I do not see any %s here.\r\n", argument );

    return;
}

void do_weigh( CHAR_DATA *ch, char *argument )
{
    CHAR_DATA *victim;
    OBJ_DATA *obj;
	short x = 0;
	bool self = FALSE;

    if ( argument[0] == '\0'  )
		self = TRUE;

	if( self )
		victim = ch;
	else
		victim = get_char_room( ch, argument );
	
	if( victim != NULL )
	{
	if( victim != ch )
	{
		if( IS_NPC( victim ) )
		{
		send_to_char( "Hard to say, really.\r\n", ch );
		return;
		}	
	
		act( "$n is inspecting $N carefully.", ch, NULL, victim, TO_NOTVICT );
		act( "$n is inspecting you carefully.", ch, NULL, victim, TO_VICT );
	
		printf_to_char( ch, "You carefully inspect %s physique and figure %s weighs about %d pounds.\r\n", HIS_HER(victim), HE_SHE(victim), number_fuzzy(body_weight(victim)) );
		return;
	}
	else
	{
		printf_to_char( ch, "You carefully inspect your physique and figure you weighs about %d pounds.\r\n", number_fuzzy(body_weight(victim)) );
		act( "$n is inspecting $mself.", ch, NULL, NULL, TO_ROOM );
		return;
	}
	}
	
	//fixme must be in hands

    for ( obj = ch->carrying; obj != NULL; obj = obj->next_content )
    {
	if ( can_see_obj( ch, obj ) && is_name( argument, obj->name ) )
	{
		if( obj->weight > 0 )
		{
		x = number_fuzzy(obj->weight);
		
		if( x < 1 )
			x = 1;
		
		printf_to_char( ch, "You carefully inspect the %s and determine that the weight is %d pound%s.\r\n", 
			smash_article(obj->short_descr), x, x > 1 ? "s" : "");
	    }
		else
			printf_to_char( ch, "You carefully inspect the %s and determine that the weight is less than a pound.\r\n", 
				smash_article(obj->short_descr) );
		
		act( "$n carefully inspects his $o.", ch, obj, NULL, TO_NOTVICT );
		
		add_roundtime( ch, 5 );
		show_roundtime( ch, 5 );
		
		return;
	}
    }

    for ( obj = ch->in_room->contents; obj != NULL; obj = obj->next_content )
    {
	if ( can_see_obj( ch, obj ) && is_name( argument, obj->name ) )
	{
		printf_to_char( ch, "You would need to be holding the %s before gauging its weight.\r\n", smash_article(obj->short_descr) );
	    return;
	}
    }
	
	printf_to_char(ch, "I do not see any %s here.\r\n", argument );
    return;
}

short get_creature_number( const char *name )
{
	short x;
	
	for( x = 0; x < MAX_CREATURE; x++ )
	{
		if( is_name(name, creature_table[x].name ) )
		return x;
	}
	
	return -1;
}

void do_skin(CHAR_DATA *ch, char *argument)
{
	CHAR_DATA *victim;
	short n;
	extern char * const dir_name[];

    if ( ch->desc == NULL )
		return;

	if( ch->in_room->vnum == ROOM_VNUM_START )
	{
		do_skincolor( ch, argument );
		return;
	}

	if ( argument[0] == '\0'  )
    {
	send_to_char( "Skin what?\r\n", ch );
	return;
    }
	
    if ( ( victim = get_char_room( ch, argument ) ) != NULL )
    {
	
	if( ch == victim )
	{
	act( "$n looks very upset with $mself.", ch, NULL, NULL, TO_ROOM );
	printf_to_char( ch, "Skinning yourself?  It can't be that bad!\r\n"  );
	return;
	}
		
	if( !IS_DEAD(victim) )
	{
		printf_to_char( ch, "%s isn't dead yet!\r\n", capitalize(HE_SHE(victim)) );
		return;
	}
	
	n = get_creature_number( victim->name );
	
	if( !IS_NPC(victim) || n == -1 || creature_table[n].skin == NULL )
	{
		printf_to_char( ch, "You cannot skin %s.\r\n", HIM_HER(victim) );
		return;
	}
	
	if( IS_SET( victim->affected_by, AFF_SKINNED ) )
	{
	printf_to_char( ch, "Already skinned.\r\n"  );
	return;
	}
	
	if( victim != ch )
	{
		int skill = 0;
		
		if( IS_NPC(ch) )
			skill = ch->level*2;
		else
			skill = skill_bonus( (ch->pcdata->learned[gsn_first_aid] + ch->pcdata->learned[gsn_survival]) /2 ) + stat_bonus( get_curr_Dexterity( ch ), ch, STAT_DEXTERITY);
		
	skill -= victim->level;
		
	if( skill > open_1d100() )
	{
	char buf[MAX_STRING_LENGTH];
	OBJ_DATA *skin;
	
	act( "$n skins $N.", ch, NULL, victim, TO_NOTVICT );
	act( "$n skins you.", ch, NULL, victim, TO_VICT );

	printf_to_char( ch, "You skin %s, producing %s.\r\n", HIM_HER(victim), creature_table[n].skin );
	SET_BIT( victim->affected_by, AFF_SKINNED );
	
	skin = create_object( get_obj_index( OBJ_VNUM_SKIN ), 0 );
	
	//fixme
	skin->cost = victim->level*25 + number_range(0,victim->level);
	
	sprintf(buf, "%s", creature_table[ n ].skin );
	
	skin->name = str_dup( buf );
	
	sprintf(buf, "%s %s", 
	select_a_an( creature_table[ n ].skin ),
	creature_table[ n ].skin );
	
	skin->short_descr = str_dup( buf );
	
	obj_to_room( skin, victim->in_room );
	}
	else
	{
	act( "$n tries to skin $N but botches the job.", ch, NULL, victim, TO_NOTVICT );
	act( "$n tries to skin you but botches the job.", ch, NULL, victim, TO_VICT );
	
	printf_to_char( ch, "You fail to skin %s and ruin the %s.\r\n", HIM_HER(victim), creature_table[n].skin );
	SET_BIT( victim->affected_by, AFF_SKINNED );
	
	}
	
	return;
	}

	return;
    }

	
	printf_to_char(ch, "I do not see any %s here.\r\n", argument );

    return;
}



void do_open( CHAR_DATA *ch, char *argument )
{
    char arg[MAX_INPUT_LENGTH];
    OBJ_DATA *obj;
    one_argument( argument, arg );

    if ( arg[0] == '\0' )
    {
	send_to_char( "Open what?\r\n", ch );
	return;
    }

    if ( ( obj = get_obj_here( ch, arg ) ) != NULL )
    {
	/* 'open object' */
	if ( obj->item_type != ITEM_CONTAINER && obj->item_type != ITEM_DOOR && obj->item_type != ITEM_FURNITURE  )
	    { send_to_char( "You can't open that.\r\n", ch ); return; }
	if ( !IS_SET(obj->value[1], CONT_CLOSED) )
	    { send_to_char( "It's already open.\r\n",      ch ); return; }
	if ( !IS_SET(obj->value[1], CONT_CLOSEABLE) )
	    { send_to_char( "You can't do that.\r\n",      ch ); return; }
	if ( IS_SET(obj->value[1], CONT_LOCKED) )
	    { send_to_char( "It's locked.\r\n",            ch ); return; }

	REMOVE_BIT(obj->value[1], CONT_CLOSED);
	printf_to_char(ch, "You open %s.\r\n", obj->short_descr );
	act( "$n opens $p.", ch, obj, NULL, TO_ROOM );
	
	if( obj->item_type == ITEM_DOOR )
	{	
		ROOM_INDEX_DATA *to_room = get_room_index( obj->value[0] );
		OBJ_DATA *room_obj;
		ROOM_INDEX_DATA *orig_room = ch->in_room;
	
		if ( to_room == NULL )
		return;
	
		for ( room_obj = to_room->contents; room_obj != NULL; room_obj = room_obj->next_content )
		{
			if ( room_obj->item_type == ITEM_DOOR && get_room_index( room_obj->value[0] ) == ch->in_room )
			{
				REMOVE_BIT(room_obj->value[1], CONT_CLOSED);
				
				char_from_room( ch );
				char_to_room( ch, to_room );

				act("$p swings open.", ch, obj, NULL, TO_ROOM );

				char_from_room( ch );
				char_to_room( ch, orig_room );
			}
		}
	}
	return;
    }
	
	printf_to_char(ch, "I do not see any %s here to open.\r\n", argument );

    return;
}



void do_close( CHAR_DATA *ch, char *argument )
{
    char arg[MAX_INPUT_LENGTH];
    OBJ_DATA *obj;

    one_argument( argument, arg );

    if ( arg[0] == '\0' )
    {
	send_to_char( "Close what?\r\n", ch );
	return;
    }

    if ( ( obj = get_obj_here( ch, arg ) ) != NULL )
    {
	/* 'close object' */
	if ( obj->item_type != ITEM_CONTAINER && obj->item_type != ITEM_DOOR && obj->item_type != ITEM_FURNITURE )
	    { send_to_char( "You can't close that.\r\n", ch ); return; }
	if ( IS_SET(obj->value[1], CONT_CLOSED) )
	    { send_to_char( "It's already closed.\r\n",    ch ); return; }
	if ( !IS_SET(obj->value[1], CONT_CLOSEABLE) )
	    { send_to_char( "You can't do that.\r\n",      ch ); return; }

	SET_BIT(obj->value[1], CONT_CLOSED);
	printf_to_char(ch, "You close %s.\r\n", obj->short_descr );
	act( "$n closes $p.", ch, obj, NULL, TO_ROOM );
	
	if( obj->item_type == ITEM_DOOR )
	{	
		ROOM_INDEX_DATA *to_room = get_room_index( obj->value[0] );
		OBJ_DATA *room_obj;
		ROOM_INDEX_DATA *orig_room = ch->in_room;
	
		if ( to_room == NULL )
		return;
	
		for ( room_obj = to_room->contents; room_obj != NULL; room_obj = room_obj->next_content )
		{
			if ( room_obj->item_type == ITEM_DOOR && get_room_index( room_obj->value[0] ) == ch->in_room )
			{
				SET_BIT(room_obj->value[1], CONT_CLOSED);
				
				char_from_room( ch );
				char_to_room( ch, to_room );

				act("$p swings shut.", ch, obj, NULL, TO_ROOM );

				char_from_room( ch );
				char_to_room( ch, orig_room );
			}
		}
	}
	
	return;
    }
	
	
	printf_to_char(ch, "I do not see any %s here to close.\r\n", argument );

    return;
}



bool has_key( CHAR_DATA *ch, int key )
{
    OBJ_DATA *obj;

    for ( obj = ch->carrying; obj != NULL; obj = obj->next_content )
    {
	if ( obj->pIndexData->vnum == key )
	    return TRUE;
    }

    return FALSE;
}



void do_lock( CHAR_DATA *ch, char *argument )
{
    char arg[MAX_INPUT_LENGTH];
    OBJ_DATA *obj;

    one_argument( argument, arg );

    if ( arg[0] == '\0' )
    {
	send_to_char( "Lock what?\r\n", ch );
	return;
    }

    if ( ( obj = get_obj_here( ch, arg ) ) != NULL )
    {
	/* 'lock object' */
	if ( obj->item_type != ITEM_CONTAINER && obj->item_type != ITEM_DOOR && obj->item_type != ITEM_FURNITURE )
	    { send_to_char( "You cannot lock that.\r\n", ch ); return; }
	if ( !IS_SET(obj->value[1], CONT_CLOSED) )
	    { send_to_char( "It is not closed.\r\n",        ch ); return; }
	if ( obj->value[2] < 0 )
	    { send_to_char( "It cannot be locked.\r\n",     ch ); return; }
	if ( !has_key( ch, obj->value[2] ) )
	    { send_to_char( "You need the key.\r\n",       ch ); return; }
	if ( IS_SET(obj->value[1], CONT_LOCKED) )
	    { send_to_char( "It is already locked.\r\n",    ch ); return; }

	SET_BIT(obj->value[1], CONT_LOCKED);
	printf_to_char(ch, "You hear a click as you lock %s.\r\n", obj->short_descr );
	act( "$n locks $p.", ch, obj, NULL, TO_ROOM );
	
		if( obj->item_type == ITEM_DOOR )
	{	
		ROOM_INDEX_DATA *to_room = get_room_index( obj->value[0] );
		OBJ_DATA *room_obj;
		ROOM_INDEX_DATA *orig_room = ch->in_room;
	
		if ( to_room == NULL )
		return;
	
		for ( room_obj = to_room->contents; room_obj != NULL; room_obj = room_obj->next_content )
		{
			if ( room_obj->item_type == ITEM_DOOR && get_room_index( room_obj->value[0] ) == ch->in_room )
			{
				SET_BIT(room_obj->value[1], CONT_LOCKED);
				
				char_from_room( ch );
				char_to_room( ch, to_room );

				act("$p clicks.", ch, obj, NULL, TO_ROOM );

				char_from_room( ch );
				char_to_room( ch, orig_room );
			}
		}
	}
	
	return;
    }
	
	printf_to_char(ch, "I do not see any %s here to lock.\r\n", argument );

    return;
}



void do_unlock( CHAR_DATA *ch, char *argument )
{
    char arg[MAX_INPUT_LENGTH];
    OBJ_DATA *obj;

    one_argument( argument, arg );

    if ( arg[0] == '\0' )
    {
	send_to_char( "Unlock what?\r\n", ch );
	return;
    }

    if ( ( obj = get_obj_here( ch, arg ) ) != NULL )
    {
	/* 'unlock object' */
	if ( obj->item_type != ITEM_CONTAINER && obj->item_type != ITEM_DOOR && obj->item_type != ITEM_FURNITURE  )
	    { send_to_char( "You cannot unlock that.\r\n", ch ); return; }
	if ( !IS_SET(obj->value[1], CONT_CLOSED) )
	    { send_to_char( "It is not closed.\r\n",        ch ); return; }
	if ( obj->value[2] < 0 )
	    { send_to_char( "It cannot be unlocked.\r\n",   ch ); return; }
	if ( !has_key( ch, obj->value[2] ) )
	    { send_to_char( "You need the key.\r\n",       ch ); return; }
	if ( !IS_SET(obj->value[1], CONT_LOCKED) )
	    { send_to_char( "It is already unlocked.\r\n",  ch ); return; }

	REMOVE_BIT(obj->value[1], CONT_LOCKED);
	printf_to_char(ch, "You hear a click as you unlock %s.\r\n", obj->short_descr );
	act( "$n unlocks $p.", ch, obj, NULL, TO_ROOM );
	
	if( obj->item_type == ITEM_DOOR )
	{	
		ROOM_INDEX_DATA *to_room = get_room_index( obj->value[0] );
		OBJ_DATA *room_obj;
		ROOM_INDEX_DATA *orig_room = ch->in_room;
	
		if ( to_room == NULL )
		return;
	
		for ( room_obj = to_room->contents; room_obj != NULL; room_obj = room_obj->next_content )
		{
			if ( room_obj->item_type == ITEM_DOOR && get_room_index( room_obj->value[0] ) == ch->in_room )
			{
				REMOVE_BIT(room_obj->value[1], CONT_LOCKED);
				
				char_from_room( ch );
				char_to_room( ch, to_room );

				act("$p clicks.", ch, obj, NULL, TO_ROOM );

				char_from_room( ch );
				char_to_room( ch, orig_room );
			}
		}
	}
	
	return;
    }
	
	printf_to_char(ch, "I do not see any %s here to unlock.\r\n", argument );

    return;
}

void do_read( CHAR_DATA *ch, char *argument )
{
	do_look( ch, argument );
    return;
}


//fixme -- injury, removal of slot rather than destruction..black vs white??
void do_invoke( CHAR_DATA *ch, char *argument )
{
	short sn = 0;
	OBJ_DATA *scroll = get_eq_char( ch, EQUIP_RIGHTHAND );
	
	if( IS_NPC(ch) )
		return;

	if( scroll == NULL || scroll->item_type != ITEM_SCROLL )
		scroll = get_eq_char( ch, EQUIP_LEFTHAND );
	
	if( scroll == NULL || scroll->item_type != ITEM_SCROLL )
	{
	    printf_to_char( ch, "You have no scroll in your hands.\r\n" );		
		return;
	}
	
	if ( !argument )
	{
		printf_to_char( ch, "You must specify a spell to invoke by number or mnemonic.\r\n" );		
		return;
	}

	if ( ( sn = spell_lookup( argument ) ) < 0 || ( spell_table[sn].spell_fun == spell_null ) )
	{
	    printf_to_char( ch, "There is no such spell.\r\n" );		
		return;
	}
	
	if ( spell_table[sn].number != scroll->value[0] && spell_table[sn].number != scroll->value[1] && spell_table[sn].number != scroll->value[2] && spell_table[sn].number != scroll->value[3] )
	{
	    printf_to_char( ch, "But %s does not contain the spell of %s.\r\n", scroll->short_descr, spell_table[sn].name );		
		return;
	}
	
	if ( ch->prepared_spell != 0 )
	{
	    printf_to_char( ch, "Your mind is too occupied to handle another spell.\r\n" );		
		return;
	}
	
	if( skill_bonus(ch->pcdata->learned[gsn_arcane_symbols]) < open_1d100() )
	{
	printf_to_char( ch, "You raise the %s and gesture to invoke the %s spell, but something doesn't work.\r\n", smash_article(scroll->short_descr),
		spell_table[sn].name );	
    act( "$n raises $o and gestures, but something doesn't work.", ch, scroll, NULL, TO_ROOM );
	
	add_roundtime( ch, 3 );
	show_roundtime( ch, 3);
	return;
	}
	
	printf_to_char( ch, "You raise the %s and gesture to invoke the %s spell.\r\n", smash_article(scroll->short_descr),
		spell_table[sn].name );	
    act( "$n raises $o and gestures.", ch, scroll, NULL, TO_ROOM );
		
	send_to_char( "Your spell is ready to cast.\r\n", ch );
	
	if( scroll->value[0] == spell_table[sn].number )
	    ch->prepared_spell = sn;
	else if( scroll->value[1] == spell_table[sn].number )
	    ch->prepared_spell = sn;
	else if( scroll->value[2] == spell_table[sn].number )
	    ch->prepared_spell = sn;
	else if( scroll->value[3] == spell_table[sn].number )
	    ch->prepared_spell = sn;
	
	add_roundtime( ch, 3 );
	show_roundtime( ch, 3 );

	if( open_1d100() < 5 )
	{
	act( "$o is consumed in a shimmering blue flame.", ch, scroll, NULL, TO_CHAR );
	act( "$o is consumed in a shimmering blue flame.", ch, scroll, NULL, TO_ROOM );
    extract_obj( scroll );
	return;
	}
	
    return;
}

void drag_obj( CHAR_DATA *ch, OBJ_DATA *obj, char *argument )
{
	extern char * const dir_name[];
	OBJ_DATA *portal;
	ROOM_INDEX_DATA *to_room;
    EXIT_DATA *pexit;
	int direction;
	int success;
	
	if( get_eq_char( ch, EQUIP_RIGHTHAND ) != NULL &&  get_eq_char( ch, EQUIP_LEFTHAND ) != NULL )
    {
	send_to_char( "You need a free hand.\r\n", ch );
	return;
    }
	
	if ( !CAN_WEAR(obj, ITEM_EQUIP_TAKE) || obj->weight > 1000 )
	{
	send_to_char( "Try as you might, it just won't budge.\r\n", ch );
	return;
	}
	
	success = body_weight(ch) + get_curr_Strength(ch) - obj->weight - get_encumbrance_level(ch);
	
	if( open_1d100() > success )
	{
		act("You struggle to drag $P.", ch, NULL, obj, TO_CHAR  );
		act("$n struggles to drag $P.", ch, NULL, obj, TO_ROOM );
		
		add_roundtime( ch, 5 );
		show_roundtime( ch, 5 );
		return;
	}
		
	portal = get_obj_list( ch, argument, ch->in_room->contents );

    if( portal == NULL )
	{
		if ( (direction = get_direction( argument )) == -1
		|| (pexit = ch->in_room->exit[direction] ) == NULL
		|| (to_room = pexit->to_room) == NULL
		|| to_room == ch->in_room )
		{
		send_to_char( "Where do you intend to drag it?\n\r", ch );
		return;
		}

		act("You drag $P $t.", ch, dir_name[direction], obj, TO_CHAR  );
		act("$n drags $P $t.", ch, dir_name[direction], obj, TO_ROOM );

		obj_from_room( obj );
		obj_to_room( obj, to_room );
		char_from_room( ch );
		char_to_room( ch, to_room );
		act("$n drags $p in.", ch, obj, NULL, TO_NOTVICT );
	}
	else
	{
		if ( portal->item_type != ITEM_PORTAL && portal->item_type != ITEM_BUILDING
		&& portal->item_type != ITEM_DOOR && portal->item_type != ITEM_CLIMB && portal->item_type != ITEM_SWIM)
		{
		send_to_char("You can't go there.\r\n", ch );
		return;
		}
	
		if ( (to_room = get_room_index( portal->value[0] )) == NULL )
		{
		send_to_char("It doesn't appear to lead anywhere.\r\n", ch );
		return;
		}
	
		act("You drag $P through $p.", ch, portal, obj, TO_CHAR  );
		act("$n drags $P through $p.", ch, portal, obj, TO_ROOM );

		obj_from_room( obj );
		obj_to_room( obj, to_room );
		char_from_room( ch );
		char_to_room( ch, to_room );
		act("$n drags $P in.", ch, NULL, obj, TO_NOTVICT );
	}

    do_look( ch, "" );
	
	{
	int rt = 10;
	
	rt -= success / 10;
	
	if( rt > 10 )
		rt = 10;
	
	if (rt > 0 )
	{
	add_roundtime( ch, rt );
	show_roundtime( ch, rt );
	}
	}
	
    return;
}

void do_drag(CHAR_DATA *ch, char *argument )
{
    extern char * const dir_name[];
    int direction;
    CHAR_DATA *victim;
    char arg1[MAX_INPUT_LENGTH];
    ROOM_INDEX_DATA *to_room;
    EXIT_DATA *pexit;
	OBJ_DATA *portal;
	OBJ_DATA *obj;
	int success;
	
    argument = one_argument( argument, arg1 );
	
	obj = get_obj_list( ch, arg1, ch->in_room->contents );
	
	if( obj != NULL )
	{
	drag_obj( ch, obj, argument );
	return;
	}

    if ( arg1 == NULL || arg1[0] == '\0' )
    {
	send_to_char("Drag whom or what where?\n\r", ch );
	return;
    }

    if ( (victim = get_char_room( ch, arg1 )) == NULL )
    {
	send_to_char( "You don't see them here.\n\r", ch );
	return;
    }
	
    if ( ch == victim )
    {
	send_to_char( "Try walking.\n\r", ch );
	return;
    }
	
	if( get_eq_char( ch, EQUIP_RIGHTHAND ) != NULL &&  get_eq_char( ch, EQUIP_LEFTHAND ) != NULL )
    {
	send_to_char( "You need a free hand.\r\n", ch );
	return;
    }

    if ( IS_NPC(victim) || !IS_DEAD(victim) )
    {
	send_to_char( "You better not.\n\r", ch );
	return;
    }

    if ( victim->position != POS_PRONE )
    {
    send_to_char( "It's difficult to drag someone that isn't lying down.\n\r", ch );
	return;
    }
	
	success = body_weight(ch) + get_curr_Strength(ch) - body_weight(victim) - get_encumbrance_level(ch) - get_encumbrance_level(victim);
	
	if( open_1d100() > success )
	{
		act("You struggle to drag $P.", ch, NULL, obj, TO_CHAR  );
		act("$n struggles to drag $P.", ch, NULL, obj, TO_ROOM );
		
		add_roundtime( ch, 5 );
		show_roundtime( ch, 5 );
		return;
	}
	
	portal = get_obj_list( ch, argument, ch->in_room->contents );

    if( portal == NULL )
	{
    if ( (direction = get_direction( argument )) == -1
    || (pexit = ch->in_room->exit[direction] ) == NULL
    || (to_room = pexit->to_room) == NULL
    || to_room == ch->in_room )
    {
	send_to_char( "Where do you intend to drag them?\n\r", ch );
	return;
    }

    act("You drag $N $t.", ch, dir_name[direction], victim, TO_CHAR  );

    act("$n drags $N $t.", ch, dir_name[direction], victim, TO_ROOM );

    char_from_room( victim );
    char_to_room( victim, to_room );
	char_from_room( ch );
    char_to_room( ch, to_room );
    act("$n drags $N in.", ch, NULL, victim, TO_NOTVICT );
	}
	else
	{
    if ( portal->item_type != ITEM_PORTAL && portal->item_type != ITEM_BUILDING
		&& portal->item_type != ITEM_DOOR && portal->item_type != ITEM_CLIMB && portal->item_type != ITEM_SWIM)
    {
	send_to_char("You can't go there.\r\n", ch );
	return;
    }

    if ( (to_room = get_room_index( portal->value[0] )) == NULL )
    {
	send_to_char("It doesn't appear to lead anywhere.\r\n", ch );
	return;
    }
	
	act("You drag $N through $p.", ch, portal, victim, TO_CHAR  );
    act("$n drags you through $p.", ch, portal, victim, TO_VICT );
    act("$n drags $N through $p.", ch, portal, victim, TO_NOTVICT  );

    char_from_room( victim );
    char_to_room( victim, to_room );
	char_from_room( ch );
    char_to_room( ch, to_room );
    act("$n drags $N in.", ch, NULL, victim, TO_NOTVICT );
	}

    do_look( ch, "" );
	do_look( victim, "" );
	
	{
	int rt = 10;
	
	rt -= success / 10;
	
	if( rt > 10 )
		rt = 10;
	
	if (rt > 0 )
	{
	add_roundtime( ch, rt );
	show_roundtime( ch, rt );
	}
	}
	
    return;
}




// Things to do:

// advance should wipe skills and add ptp/mtp

#if defined(macintosh)
#include <types.h>
#else
#include <sys/types.h>
#endif
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <time.h>
#include "merc.h"



/*
 * Local functions.
 */
ROOM_INDEX_DATA *	find_location	args( ( CHAR_DATA *ch, char *arg ) );



void do_wizhelp( CHAR_DATA *ch, char *argument )
{
    char buf[MAX_STRING_LENGTH];
    int cmd;
    int col;

    col = 0;
    for ( cmd = 0; cmd_table[cmd].name[0] != '\0'; cmd++ )
    {
        if ( cmd_table[cmd].level >= LEVEL_HERO
        &&   cmd_table[cmd].level <= get_trust( ch ) )
	{
	    sprintf( buf, "%-12s", cmd_table[cmd].name );
	    send_to_char( buf, ch );
	    if ( ++col % 6 == 0 )
		send_to_char( "\r\n", ch );
	}
    }

    if ( col % 6 != 0 )
	send_to_char( "\r\n", ch );
    return;
}



void do_bamfin( CHAR_DATA *ch, char *argument )
{
    if ( !IS_NPC(ch) )
    {
	smash_tilde( argument );
	free_string( ch->pcdata->bamfin );
	ch->pcdata->bamfin = str_dup( argument );
	send_to_char( "Ok.\r\n", ch );
    }
    return;
}



void do_bamfout( CHAR_DATA *ch, char *argument )
{
    if ( !IS_NPC(ch) )
    {
	smash_tilde( argument );
	free_string( ch->pcdata->bamfout );
	ch->pcdata->bamfout = str_dup( argument );
	send_to_char( "Ok.\r\n", ch );
    }
    return;
}



void do_deny( CHAR_DATA *ch, char *argument )
{
    char arg[MAX_INPUT_LENGTH];
    CHAR_DATA *victim;

    one_argument( argument, arg );
    if ( arg[0] == '\0' )
    {
	send_to_char( "Deny whom?\r\n", ch );
	return;
    }

    if ( ( victim = get_char_world( ch, arg ) ) == NULL )
    {
	send_to_char( "They aren't here.\r\n", ch );
	return;
    }

    if ( IS_NPC(victim) )
    {
	send_to_char( "Not on NPC's.\r\n", ch );
	return;
    }

    if ( get_trust( victim ) >= get_trust( ch ) )
    {
	send_to_char( "You failed.\r\n", ch );
	return;
    }

    SET_BIT(victim->act, PLR_DENY);
    send_to_char( "You have been locked out.\r\n", victim );
    send_to_char( "OK.\r\n", ch );
    do_quit( victim, "" );

    return;
}



void do_disconnect( CHAR_DATA *ch, char *argument )
{
    char arg[MAX_INPUT_LENGTH];
    DESCRIPTOR_DATA *d;
    CHAR_DATA *victim;

    one_argument( argument, arg );
    if ( arg[0] == '\0' )
    {
	send_to_char( "Disconnect whom?\r\n", ch );
	return;
    }

    if ( ( victim = get_char_world( ch, arg ) ) == NULL )
    {
	send_to_char( "They aren't here.\r\n", ch );
	return;
    }

    if ( victim->desc == NULL )
    {
	act( "$N doesn't have a descriptor.", ch, NULL, victim, TO_CHAR );
	return;
    }

    for ( d = descriptor_list; d != NULL; d = d->next )
    {
	if ( d == victim->desc )
	{
	    close_socket( d );
	    send_to_char( "Ok.\r\n", ch );
	    return;
	}
    }

    bug( "Do_disconnect: desc not found.", 0 );
    send_to_char( "Descriptor not found!\r\n", ch );
    return;
}



void do_pardon( CHAR_DATA *ch, char *argument )
{
	//fixme
    return;
}



void do_echo( CHAR_DATA *ch, char *argument )
{
    DESCRIPTOR_DATA *d;

    if ( argument[0] == '\0' )
    {
	send_to_char( "Echo what?\r\n", ch );
	return;
    }

    for ( d = descriptor_list; d; d = d->next )
    {
	if ( d->connected == CON_PLAYING )
	{
	    send_to_char( argument, d->character );
	    send_to_char( "\r\n",   d->character );
	}
    }

    return;
}



void do_recho( CHAR_DATA *ch, char *argument )
{
    DESCRIPTOR_DATA *d;

    if ( argument[0] == '\0' )
    {
	send_to_char( "Recho what?\r\n", ch );
	return;
    }

    for ( d = descriptor_list; d; d = d->next )
    {
	if ( d->connected == CON_PLAYING
	&&   d->character->in_room == ch->in_room )
	{
	    send_to_char( argument, d->character );
	    send_to_char( "\r\n",   d->character );
	}
    }

    return;
}



ROOM_INDEX_DATA *find_location( CHAR_DATA *ch, char *arg )
{
    CHAR_DATA *victim;
    OBJ_DATA *obj;

    if ( is_number(arg) )
	return get_room_index( atoi( arg ) );

    if ( ( victim = get_char_world( ch, arg ) ) != NULL )
	return victim->in_room;

    if ( ( obj = get_obj_world( ch, arg ) ) != NULL )
	return obj->in_room;

    return NULL;
}



void do_transfer( CHAR_DATA *ch, char *argument )
{
    char arg1[MAX_INPUT_LENGTH];
    char arg2[MAX_INPUT_LENGTH];
    ROOM_INDEX_DATA *location;
    DESCRIPTOR_DATA *d;
    CHAR_DATA *victim;

    argument = one_argument( argument, arg1 );
    argument = one_argument( argument, arg2 );

    if ( arg1[0] == '\0' )
    {
	send_to_char( "Transfer whom (and where)?\r\n", ch );
	return;
    }

    if ( !str_cmp( arg1, "all" ) )
    {
	for ( d = descriptor_list; d != NULL; d = d->next )
	{
	    if ( d->connected == CON_PLAYING
	    &&   d->character != ch
	    &&   d->character->in_room != NULL
	    &&   can_see( ch, d->character ) )
	    {
		char buf[MAX_STRING_LENGTH];
		sprintf( buf, "%s %s", d->character->name, arg2 );
		do_transfer( ch, buf );
	    }
	}
	return;
    }

    /*
     * Thanks to Grodyn for the optional location parameter.
     */
    if ( arg2[0] == '\0' )
    {
	location = ch->in_room;
    }
    else
    {
	if ( ( location = find_location( ch, arg2 ) ) == NULL )
	{
	    send_to_char( "No such location.\r\n", ch );
	    return;
	}

	if ( room_is_private( location ) )
	{
	    send_to_char( "That room is private right now.\r\n", ch );
	    return;
	}
    }

    if ( ( victim = get_char_world( ch, arg1 ) ) == NULL )
    {
	send_to_char( "They aren't here.\r\n", ch );
	return;
    }

    if ( victim->in_room == NULL )
    {
	send_to_char( "They are in limbo.\r\n", ch );
	return;
    }

    act( "$n suddenly disappears.", victim, NULL, NULL, TO_ROOM );
    char_from_room( victim );
    char_to_room( victim, location );
    act( "$n suddenly appears.", victim, NULL, NULL, TO_ROOM );
    if ( ch != victim )
	act( "$n has transferred you.", ch, NULL, victim, TO_VICT );
    do_look( victim, "auto" );
    send_to_char( "Ok.\r\n", ch );
}


void do_at( CHAR_DATA *ch, char *argument )
{
    char arg[MAX_INPUT_LENGTH];
    ROOM_INDEX_DATA *location;
    ROOM_INDEX_DATA *original;
    CHAR_DATA *wch;
    
    argument = one_argument( argument, arg );

    if ( arg[0] == '\0' || argument[0] == '\0' )
    {
	send_to_char( "At where what?\n\r", ch );
	return;
    }

    if ( ( location = find_location( ch, arg ) ) == NULL )
    {
	send_to_char( "No such location.\n\r", ch );
	return;
    }

    if ( room_is_private( location ) )
    {
	send_to_char( "That room is private right now.\n\r", ch );
	return;
    }

    original = ch->in_room;
    char_from_room( ch );
    char_to_room( ch, location );
    interpret( ch, argument );

    /*
     * See if 'ch' still exists before continuing!
     * Handles 'at XXXX quit' case.
     */
    for ( wch = char_list; wch != NULL; wch = wch->next )
    {
	if ( wch == ch )
	{
	    char_from_room( ch );
	    char_to_room( ch, original );
	    break;
	}
    }

    return;
}



void do_goto( CHAR_DATA *ch, char *argument )
{
    char arg[MAX_INPUT_LENGTH];
    ROOM_INDEX_DATA *location;

    one_argument( argument, arg );
    if ( arg[0] == '\0' )
    {
	send_to_char( "Goto where?\r\n", ch );
	return;
    }

    if ( ( location = find_location( ch, arg ) ) == NULL )
    {
	send_to_char( "No such location.\r\n", ch );
	return;
    }

    if ( room_is_private( location ) )
    {
	send_to_char( "That room is private right now.\r\n", ch );
	return;
    }

    if ( !IS_SET(ch->act, PLR_WIZINVIS) )
    {
	act( "$n $T.", ch, NULL,
	    (ch->pcdata != NULL && ch->pcdata->bamfout[0] != '\0')
	    ? ch->pcdata->bamfout : "disappears",  TO_ROOM );
    }

    char_from_room( ch );
    char_to_room( ch, location );

    if ( !IS_SET(ch->act, PLR_WIZINVIS) )
    {
	act( "$n $T.", ch, NULL,
	    (ch->pcdata != NULL && ch->pcdata->bamfin[0] != '\0')
	    ? ch->pcdata->bamfin : "appears", TO_ROOM );
    }

    do_look( ch, "auto" );
    return;
}



void do_rstat( CHAR_DATA *ch, char *argument )
{
    char buf[MAX_STRING_LENGTH];
    char arg[MAX_INPUT_LENGTH];
    ROOM_INDEX_DATA *location;
    OBJ_DATA *obj;
    CHAR_DATA *rch;
    int door;

    one_argument( argument, arg );
    location = ( arg[0] == '\0' ) ? ch->in_room : find_location( ch, arg );
    if ( location == NULL )
    {
	send_to_char( "No such location.\r\n", ch );
	return;
    }

    if ( ch->in_room != location && room_is_private( location ) )
    {
	send_to_char( "That room is private right now.\r\n", ch );
	return;
    }

    sprintf( buf, "Name: '%s.'\r\nArea: '%s'.\r\n",
	location->name,
	location->area->name );
    send_to_char( buf, ch );

    sprintf( buf,
	"Vnum: %d.  Sector: %d.  Light: %d.\r\n",
	location->vnum,
	location->sector_type,
	location->light );
    send_to_char( buf, ch );
	
	if ( location->spec_fun != NULL )
    {
	send_to_char( "Room has spec fun.\n\r", ch );
	//if (spec_temple( ch, NULL, NULL, NULL, ch->in_room ) )
	//	      if ( (*obj->spec_fun) ( NULL, NULL, NULL, NULL, obj ) )
	if ( (*location->spec_fun)( ch, NULL, NULL, NULL, ch->in_room ) )
		send_to_char( "room is true\n\r", ch );
    }

    sprintf( buf,
	"Room flags: %d.\r\nDescription:\r\n%s",
	location->room_flags,
	location->description );
    send_to_char( buf, ch );

    if ( location->extra_descr != NULL )
    {
	EXTRA_DESCR_DATA *ed;

	send_to_char( "Extra description keywords: '", ch );
	for ( ed = location->extra_descr; ed; ed = ed->next )
	{
	    send_to_char( ed->keyword, ch );
	    if ( ed->next != NULL )
		send_to_char( " ", ch );
	}
	send_to_char( "'.\r\n", ch );
    }

    send_to_char( "Characters:", ch );
    for ( rch = location->people; rch; rch = rch->next_in_room )
    {
	send_to_char( " ", ch );
	one_argument( rch->name, buf );
	send_to_char( buf, ch );
    }

    send_to_char( ".\r\nObjects:   ", ch );
    for ( obj = location->contents; obj; obj = obj->next_content )
    {
	send_to_char( " ", ch );
	one_argument( obj->name, buf );
	send_to_char( buf, ch );
    }
    send_to_char( ".\r\n", ch );

    for ( door = 0; door <= 10; door++ )
    {
	EXIT_DATA *pexit;

	if ( ( pexit = location->exit[door] ) != NULL )
	{
	    sprintf( buf,
		"Door: %d.  To: %d.  Key: %d.  Exit flags: %d.\r\nKeyword: '%s'.  Description: %s",

		door,
		pexit->to_room != NULL ? pexit->to_room->vnum : 0,
	    	pexit->key,
	    	pexit->exit_info,
	    	pexit->keyword,
	    	pexit->description[0] != '\0'
		    ? pexit->description : "(none).\r\n" );
	    send_to_char( buf, ch );
	}
    }

    return;
}



void do_ostat( CHAR_DATA *ch, char *argument )
{
    char buf[MAX_STRING_LENGTH];
    char arg[MAX_INPUT_LENGTH];
    AFFECT_DATA *paf;
    OBJ_DATA *obj;

    one_argument( argument, arg );

    if ( arg[0] == '\0' )
    {
	send_to_char( "Ostat what?\r\n", ch );
	return;
    }

    if ( ( obj = get_obj_world( ch, arg ) ) == NULL )
    {
	send_to_char( "Nothing like that in hell, earth, or heaven.\r\n", ch );
	return;
    }

    sprintf( buf, "Name: %s.\r\n",
	obj->name );
    send_to_char( buf, ch );

    sprintf( buf, "Vnum: %d.  Type: %s.\r\n",
	obj->pIndexData->vnum, item_type_name( obj ) );
    send_to_char( buf, ch );

    sprintf( buf, "Short description: %s.\r\nLong description: %s\r\n",
	obj->short_descr, obj->description );
    send_to_char( buf, ch );

    sprintf( buf, "Wear bits: %ld.  Extra bits: %s.\r\n",
	obj->wear_flags, extra_bit_name( obj->extra_flags ) );
    send_to_char( buf, ch );

    sprintf( buf, "Number: %d/%d.  Weight: %d/%d.\r\n",
	1,           get_obj_number( obj ),
	obj->weight, get_obj_weight( obj ) );
    send_to_char( buf, ch );

    sprintf( buf, "Cost: %d.  Timer: %d.  Level: %d.\r\n",
	obj->cost, obj->timer, obj->level );
    send_to_char( buf, ch );

    sprintf( buf,
	"In room: %d.  In object: %s.  Carried by: %s.  Wear_loc: %d.\r\n",
	obj->in_room    == NULL    ?        0 : obj->in_room->vnum,
	obj->in_obj     == NULL    ? "(none)" : obj->in_obj->short_descr,
	obj->carried_by == NULL    ? "(none)" : obj->carried_by->name,
	obj->wear_loc );
    send_to_char( buf, ch );
	
	printf_to_char( ch, "ST: %d DU: %d\r\n", obj->st, obj->du );

    sprintf( buf, "Values: %d %d %d %d.\r\n",
	obj->value[0], obj->value[1], obj->value[2], obj->value[3] );
    send_to_char( buf, ch );
	
	if ( obj->spec_fun != NULL )
	printf_to_char( ch, "obj has spec fun\r\n" );

	if ( obj->verb_trap != NULL )
    {
	VERB_TRAP_DATA *vt;

	for ( vt = obj->verb_trap; vt != NULL; vt = vt->next )
	{
	    printf_to_char( ch, "*%s\r\n%s\r\n%s\r\nGSL: %s\r\n", vt->verb, vt->firstPersonMessage, vt->roomMessage, vt->GSL );
		
	    if ( vt->next != NULL )
		send_to_char( "\r\n", ch );
	}

	send_to_char( "\r\n", ch );
    }
	

    if ( obj->extra_descr != NULL || obj->pIndexData->extra_descr != NULL )
    {
	EXTRA_DESCR_DATA *ed;

	send_to_char( "Extra description keywords: '", ch );

	for ( ed = obj->extra_descr; ed != NULL; ed = ed->next )
	{
	    send_to_char( ed->keyword, ch );
	    if ( ed->next != NULL )
		send_to_char( " ", ch );
	}

	for ( ed = obj->pIndexData->extra_descr; ed != NULL; ed = ed->next )
	{
	    send_to_char( ed->keyword, ch );
	    if ( ed->next != NULL )
		send_to_char( " ", ch );
	}

	send_to_char( "'.\r\n", ch );
    }

    for ( paf = obj->affected; paf != NULL; paf = paf->next )
    {
	sprintf( buf, "Obj: Affects %s by %d.\r\n",
	    affect_loc_name( paf->location ), paf->modifier );
	send_to_char( buf, ch );
	
	printf_to_char(ch, "Obj: type: %d duration: %d bitvector: %d\r\n", paf->type, paf->duration, paf->bitvector);
	
    }

    for ( paf = obj->pIndexData->affected; paf != NULL; paf = paf->next )
    {
	sprintf( buf, "Index: Affects %s by %d.\r\n",
	    affect_loc_name( paf->location ), paf->modifier );
	send_to_char( buf, ch );
	
		
	printf_to_char(ch, "Index: type: %d duration: %d bitvector: %d\r\n", paf->type, paf->duration, paf->bitvector);
	
    }

    return;
}



void do_mstat( CHAR_DATA *ch, char *argument )
{
    char buf[MAX_STRING_LENGTH];
    char arg[MAX_INPUT_LENGTH];
    AFFECT_DATA *paf;
    CHAR_DATA *victim;

    one_argument( argument, arg );

    if ( arg[0] == '\0' )
    {
	send_to_char( "Mstat whom?\r\n", ch );
	return;
    }

    if ( ( victim = get_char_world( ch, arg ) ) == NULL )
    {
	send_to_char( "They aren't here.\r\n", ch );
	return;
    }

    sprintf( buf, "Name: %s.\r\n",
	victim->name );
    send_to_char( buf, ch );

    sprintf( buf, "Vnum: %d.  Sex: %s.  Room: %d.\r\n",
	IS_NPC(victim) ? victim->pIndexData->vnum : 0,
	victim->sex == SEX_MALE    ? "male"   :
	victim->sex == SEX_FEMALE  ? "female" : "neutral",
	victim->in_room == NULL    ?        0 : victim->in_room->vnum
	);
    send_to_char( buf, ch );
	
	printf_to_char( ch, "Wait: %d.  Stun: %d  Predelay %d.\r\n", victim->wait, victim->stun, victim->predelay_time );
	
	if ( IS_NPC(victim) && victim->hunting != NULL )
    {
        printf_to_char( ch, "Hunting victim: %s (%s)\r\n",
                IS_NPC(victim->hunting) ? victim->hunting->short_descr
                                        : victim->hunting->name,
                IS_NPC(victim->hunting) ? "MOB" : "PLAYER" );
    }
	
	{
	int iBody = 1;
	
	while( iBody < MAX_BODY   )
	{
		printf_to_char( ch, "%s: injury: %3d scar: %3d bleed: %3d bandage: %3d armor: %3d\r\n", 
		
		body_name[iBody], 
		victim->body[iBody], 
		victim->scars[iBody],
		victim->bleed[iBody],
		victim->bandage[iBody],
		get_armor_hit_loc( victim, iBody ) );
	
	iBody++;
	}
	
	}

    sprintf( buf,
	"Str: %d  End: %d  Dex: %d  Spd: %d  Wil: %d\r\nPot: %d  Jdg: %d  Int: %d  Wis: %d  Cha: %d\r\n",
	get_curr_Strength(victim),
	get_curr_Endurance(victim),
	get_curr_Dexterity(victim),
	get_curr_Speed(victim),
	get_curr_Willpower(victim),
	get_curr_Potency(victim),
	get_curr_Judgement(victim),
	get_curr_Intelligence(victim),
	get_curr_Wisdom(victim),
	get_curr_Charm(victim) );
    send_to_char( buf, ch );

    sprintf( buf, "Stance: %d.  Hp: %d/%d.  Mana: %d/%d.  Move: %d/%d.  Practices: %d.\r\n",
	victim->stance,
	victim->hit,         victim->max_hit,
	victim->mana,        victim->max_mana,
	victim->move,        victim->max_move,
	victim->practice );
    send_to_char( buf, ch );
	
	printf_to_char( ch, "AS: %d.  DS: %d.  CS: %d.  TD: %d.\n\r", victim->as, victim->ds, victim->cs, victim->td );

    sprintf( buf,
	"Lv: %d.  Class: %d.  Race: %d.  Gold: %d.  Exp: %d.\r\n",
	victim->level,       victim->class,  victim->race,
	victim->gold,         victim->exp );
    send_to_char( buf, ch );

    sprintf( buf, "Position: %d.\r\n",
	victim->position );
    send_to_char( buf, ch );

    if ( !IS_NPC(victim) )
    {
	sprintf( buf,
	    "Thirst: %d.  Full: %d.  Drunk: %d.  Saving throw: %d.\r\n",
	    victim->pcdata->condition[COND_THIRST],
	    victim->pcdata->condition[COND_FULL],
	    victim->pcdata->condition[COND_DRUNK],
	    victim->saving_throw );
	send_to_char( buf, ch );
    }

    sprintf( buf, "Carry number: %d.  Carry weight: %d.\r\n",
	victim->carry_number, victim->carry_weight );
    send_to_char( buf, ch );

    sprintf( buf, "Age: %d.  Played: %d.  Timer: %d.  Act: %d.\r\n",
	get_age( victim ), (int) victim->played, victim->timer, victim->act );
    send_to_char( buf, ch );

    sprintf( buf, "Master: %s.  Leader: %s.  Affected by: %s.\r\n",
	victim->master      ? victim->master->name   : "(none)",
	victim->leader      ? victim->leader->name   : "(none)",
	affect_bit_name( victim->affected_by ) );
    send_to_char( buf, ch );

    sprintf( buf, "Short description: %s.\r\nLong  description: %s",
	victim->short_descr,
	victim->long_descr[0] != '\0' ? victim->long_descr : "(none).\r\n" );
    send_to_char( buf, ch );

    if ( IS_NPC(victim) && victim->spec_fun != 0 )
		send_to_char( "Mobile has spec fun.\r\n", ch );
	

    for ( paf = victim->affected; paf != NULL; paf = paf->next )
    {
	sprintf( buf,
	    "Spell: '%s' modifies %s by %d for %d hours with bits %s.\r\n",
	    spell_table[(int) paf->type].name,
	    affect_loc_name( paf->location ),
	    paf->modifier,
	    paf->duration,
	    affect_bit_name( paf->bitvector )
	    );
	send_to_char( buf, ch );
    }

    return;
}



void do_mfind( CHAR_DATA *ch, char *argument )
{
    extern int top_mob_index;
    char buf[MAX_STRING_LENGTH];
    char arg[MAX_INPUT_LENGTH];
    MOB_INDEX_DATA *pMobIndex;
    int vnum;
    int nMatch;
    bool fAll;
    bool found;

    one_argument( argument, arg );
    if ( arg[0] == '\0' )
    {
	send_to_char( "Mfind whom?\r\n", ch );
	return;
    }

    fAll	= !str_cmp( arg, "all" );
    found	= FALSE;
    nMatch	= 0;

    /*
     * Yeah, so iterating over all vnum's takes 10,000 loops.
     * Get_mob_index is fast, and I don't feel like threading another link.
     * Do you?
     * -- Furey
     */
    for ( vnum = 0; nMatch < top_mob_index; vnum++ )
    {
	if ( ( pMobIndex = get_mob_index( vnum ) ) != NULL )
	{
	    nMatch++;
	    if ( fAll || is_name( arg, pMobIndex->player_name ) )
	    {
		found = TRUE;
		sprintf( buf, "[%5d] %s\r\n",
		    pMobIndex->vnum, capitalize( pMobIndex->short_descr ) );
		send_to_char( buf, ch );
	    }
	}
    }

    if ( !found )
	send_to_char( "Nothing like that in hell, earth, or heaven.\r\n", ch );

    return;
}



void do_ofind( CHAR_DATA *ch, char *argument )
{
    extern int top_obj_index;
    char buf[MAX_STRING_LENGTH];
    char arg[MAX_INPUT_LENGTH];
    OBJ_INDEX_DATA *pObjIndex;
    int vnum;
    int nMatch;
    bool fAll;
    bool found;

    one_argument( argument, arg );
    if ( arg[0] == '\0' )
    {
	send_to_char( "Ofind what?\r\n", ch );
	return;
    }

    fAll	= !str_cmp( arg, "all" );
    found	= FALSE;
    nMatch	= 0;

    /*
     * Yeah, so iterating over all vnum's takes 10,000 loops.
     * Get_obj_index is fast, and I don't feel like threading another link.
     * Do you?
     * -- Furey
     */
    for ( vnum = 0; nMatch < top_obj_index; vnum++ )
    {
	if ( ( pObjIndex = get_obj_index( vnum ) ) != NULL )
	{
	    nMatch++;
	    if ( fAll || is_name( arg, pObjIndex->name ) )
	    {
		found = TRUE;
		sprintf( buf, "[%5d] %s\r\n",
		    pObjIndex->vnum, capitalize( pObjIndex->short_descr ) );
		send_to_char( buf, ch );
	    }
	}
    }

    if ( !found )
	send_to_char( "Nothing like that in hell, earth, or heaven.\r\n", ch );

    return;
}



void do_mwhere( CHAR_DATA *ch, char *argument )
{
    char buf[MAX_STRING_LENGTH];
    char arg[MAX_INPUT_LENGTH];
    CHAR_DATA *victim;
    bool found;

    one_argument( argument, arg );
    if ( arg[0] == '\0' )
    {
	send_to_char( "Mwhere whom?\r\n", ch );
	return;
    }

    found = FALSE;
    for ( victim = char_list; victim != NULL; victim = victim->next )
    {
	if ( IS_NPC(victim)
	&&   victim->in_room != NULL
	&&   is_name( arg, victim->name ) )
	{
	    found = TRUE;
	    sprintf( buf, "[%5d] %-28s [%5d] %s\r\n",
		victim->pIndexData->vnum,
		victim->short_descr,
		victim->in_room->vnum,
		victim->in_room->name );
	    send_to_char( buf, ch );
	}
    }

    if ( !found )
	act( "You didn't find any $T.", ch, NULL, arg, TO_CHAR );

    return;
}



void do_reboo( CHAR_DATA *ch, char *argument )
{
    send_to_char( "If you want to REBOOT, spell it out.\r\n", ch );
    return;
}



void do_reboot( CHAR_DATA *ch, char *argument )
{
    char buf[MAX_STRING_LENGTH];
    extern bool merc_down;

    sprintf( buf, "Reboot by %s.", ch->name );
    do_echo( ch, buf );
    merc_down = TRUE;
    return;
}



void do_shutdow( CHAR_DATA *ch, char *argument )
{
    send_to_char( "If you want to SHUTDOWN, spell it out.\r\n", ch );
    return;
}



void do_shutdown( CHAR_DATA *ch, char *argument )
{
    char buf[MAX_STRING_LENGTH];
    extern bool merc_down;
	
	do_force(ch, "all save");

    sprintf( buf, "Shutdown by %s.", ch->name );
    append_file( ch, SHUTDOWN_FILE, buf );
    strcat( buf, "\r\n" );
    do_echo( ch, buf );
    merc_down = TRUE;
    return;
}



void do_snoop( CHAR_DATA *ch, char *argument )
{
    char arg[MAX_INPUT_LENGTH];
    DESCRIPTOR_DATA *d;
    CHAR_DATA *victim;

    one_argument( argument, arg );

    if ( arg[0] == '\0' )
    {
	send_to_char( "Snoop whom?\r\n", ch );
	return;
    }

    if ( ( victim = get_char_world( ch, arg ) ) == NULL )
    {
	send_to_char( "They aren't here.\r\n", ch );
	return;
    }

    if ( victim->desc == NULL )
    {
	send_to_char( "No descriptor to snoop.\r\n", ch );
	return;
    }

    if ( victim == ch )
    {
	send_to_char( "Cancelling all snoops.\r\n", ch );
	for ( d = descriptor_list; d != NULL; d = d->next )
	{
	    if ( d->snoop_by == ch->desc )
		d->snoop_by = NULL;
	}
	return;
    }

    if ( victim->desc->snoop_by != NULL )
    {
	send_to_char( "Busy already.\r\n", ch );
	return;
    }

    if ( get_trust( victim ) >= get_trust( ch ) )
    {
	send_to_char( "You failed.\r\n", ch );
	return;
    }

    if ( ch->desc != NULL )
    {
	for ( d = ch->desc->snoop_by; d != NULL; d = d->snoop_by )
	{
	    if ( d->character == victim || d->original == victim )
	    {
		send_to_char( "No snoop loops.\r\n", ch );
		return;
	    }
	}
    }

    victim->desc->snoop_by = ch->desc;
    send_to_char( "Ok.\r\n", ch );
    return;
}



void do_switch( CHAR_DATA *ch, char *argument )
{
    char arg[MAX_INPUT_LENGTH];
    CHAR_DATA *victim;

    one_argument( argument, arg );

    if ( arg[0] == '\0' )
    {
	send_to_char( "Switch into whom?\r\n", ch );
	return;
    }

    if ( ch->desc == NULL )
	return;

    if ( ch->desc->original != NULL )
    {
	send_to_char( "You are already switched.\r\n", ch );
	return;
    }

    if ( ( victim = get_char_world( ch, arg ) ) == NULL )
    {
	send_to_char( "They aren't here.\r\n", ch );
	return;
    }

    if ( victim == ch )
    {
	send_to_char( "Ok.\r\n", ch );
	return;
    }

    if ( victim->desc != NULL )
    {
	send_to_char( "Character in use.\r\n", ch );
	return;
    }

    ch->desc->character = victim;
    ch->desc->original  = ch;
    victim->desc        = ch->desc;
    ch->desc            = NULL;
    send_to_char( "Ok.\r\n", victim );
    return;
}



void do_return( CHAR_DATA *ch, char *argument )
{
    if ( ch->desc == NULL )
	return;

    if ( ch->desc->original == NULL )
    {
	send_to_char( "You aren't switched.\r\n", ch );
	return;
    }

    send_to_char( "You return to your original body.\r\n", ch );
    ch->desc->character       = ch->desc->original;
    ch->desc->original        = NULL;
    ch->desc->character->desc = ch->desc;
    ch->desc                  = NULL;
    return;
}



void do_mload( CHAR_DATA *ch, char *argument )
{
    char arg[MAX_INPUT_LENGTH];
    MOB_INDEX_DATA *pMobIndex;
    CHAR_DATA *victim;

    one_argument( argument, arg );

    if ( arg[0] == '\0' || !is_number(arg) )
    {
	send_to_char( "Syntax: mload <vnum>.\r\n", ch );
	return;
    }

    if ( ( pMobIndex = get_mob_index( atoi( arg ) ) ) == NULL )
    {
	send_to_char( "No mob has that vnum.\r\n", ch );
	return;
    }

    victim = create_mobile( pMobIndex );
    char_to_room( victim, ch->in_room );

	
    act( "$n has created $N!", ch, NULL, victim, TO_ROOM );

    send_to_char( "Ok.\r\n", ch );
    return;
}

void do_cload( CHAR_DATA *ch, char *argument )
{
    char arg[MAX_INPUT_LENGTH];
    MOB_INDEX_DATA *pMobIndex;
    CHAR_DATA *victim;
	short num;
	char buf[MSL];
	bool fList	= FALSE;

    one_argument( argument, arg );
	
	fList = !str_cmp( arg, "list" );
	
	if( fList )
	{
		for( num = 0; num < MAX_CREATURE; num++ )
		{
			printf_to_char( ch, "%2d: (%2d) %-30s AS:%3d DS:%3d %s %d %s\r\n", num, creature_table[num].level,
				creature_table[num].name, creature_table[num].ob, creature_table[num].db,
					IS_SET(creature_table[num].act, ACT_SPELLCASTER) ? "Spellcaster" : " ___ ", 
					
						creature_table[num].class, creature_table[num].skin != NULL ? creature_table[num].skin : ""  );
		}
		
		return;
	}

    if ( arg[0] == '\0' || !is_number(arg) )
    {
	send_to_char( "Syntax: cload <creature number>.\r\n", ch );
	send_to_char( "        cload list.\r\n", ch );
	return;
    }

    if ( ( pMobIndex = get_mob_index( 2 ) ) == NULL )
    {
	send_to_char( "Can't create generic mob template.\r\n", ch );
	return;
    }
	
	num = atoi(arg);
	
	if( num < 0 || num >= MAX_CREATURE )
	{
	send_to_char( "Invalid creature number.\r\n", ch );
	return;
	}


    victim = create_mobile( pMobIndex );
		
	strcpy( buf, "" );
	sprintf(buf, "%s", creature_table[num].name );	
	victim->name = str_dup(buf);
	
	strcpy( buf, "" );
	sprintf(buf, "%s %s", select_a_an( creature_table[num].name ), creature_table[num].name );	
	victim->short_descr = str_dup( buf );

	victim->level = creature_table[num].level;
	victim->class = creature_table[num].class;
	victim->as = creature_table[num].ob;
	victim->ds = creature_table[num].db;
	
	victim->cs = creature_table[num].cs;
	victim->td = creature_table[num].td;
	
	victim->act = creature_table[num].act|ACT_IS_NPC|ACT_AGGRESSIVE;
	victim->affected_by = creature_table[num].aff;
	
	victim->max_hit = creature_table[num].hp;
	victim->hit = victim->max_hit;	
	
	victim->max_mana = victim->level * 3;
	victim->mana = victim->max_mana;
	
	victim->sex = number_range(1,2);
	
    char_to_room( victim, ch->in_room );

    act( "$n gestures oddly and $N appears in a puff of smoke.", ch, NULL, victim, TO_ROOM );

    send_to_char( "Ok.\r\n", ch );
	
	//free_string( buf );
	
    return;
}


void do_rload( CHAR_DATA *ch, char *argument )
{
    char arg1[MAX_INPUT_LENGTH];
    char arg2[MAX_INPUT_LENGTH];
    OBJ_INDEX_DATA *pObjIndex;
    OBJ_DATA *obj;
    int level;

    argument = one_argument( argument, arg1 );
    argument = one_argument( argument, arg2 );

    if ( arg1[0] == '\0' || !is_number( arg1 ) )
    {
        send_to_char( "Syntax: oload <vnum>.\r\n", ch );
        return;
    }

    if ( arg2[0] == '\0' )
    {
	level = 10;
    }
    else
    {
        level = atoi( arg2 );
		
		if( level > 10 )
			level = 10;
			
		if( level < 0 )
			level = 0;
    }

    if ( ( pObjIndex = get_obj_index( atoi( arg1 ) ) ) == NULL )
    {
	send_to_char( "No object has that vnum.\r\n", ch );
	return;
    }

    obj = create_object( pObjIndex, level );
    if ( CAN_WEAR(obj, ITEM_EQUIP_TAKE) )
    {
	obj_to_char( obj, ch );
    }
    else
    {
	obj_to_room( obj, ch->in_room );
	act( "$n has created $p!", ch, obj, NULL, TO_ROOM );
    }
    send_to_char( "Ok.\r\n", ch );
    return;
}


void do_oload( CHAR_DATA *ch, char *argument )
{
    char arg1[MAX_INPUT_LENGTH];
    char arg2[MAX_INPUT_LENGTH];
    OBJ_INDEX_DATA *pObjIndex;
    OBJ_DATA *obj;
    int level;

    argument = one_argument( argument, arg1 );
    argument = one_argument( argument, arg2 );

    if ( arg1[0] == '\0' || !is_number( arg1 ) )
    {
        send_to_char( "Syntax: oload <vnum> <level>.\r\n", ch );
        return;
    }

    if ( arg2[0] == '\0' )
    {
	level = get_trust( ch );
    }
    else
    {
	/*
	 * New feature from Alander.
	 */
        if ( !is_number( arg2 ) )
        {
	    send_to_char( "Syntax: oload <vnum> <level>.\r\n", ch );
	    return;
        }
        level = atoi( arg2 );
	if ( level < 0 || level > get_trust( ch ) )
        {
	    send_to_char( "Limited to your trust level.\r\n", ch );
	    return;
        }
    }

    if ( ( pObjIndex = get_obj_index( atoi( arg1 ) ) ) == NULL )
    {	
	send_to_char( "No object has that vnum.\r\n", ch );
	return;
    }

    obj = create_object( pObjIndex, level );
    if ( CAN_WEAR(obj, ITEM_EQUIP_TAKE) )
    {
	obj_to_char( obj, ch );
    }
    else
    {
	obj_to_room( obj, ch->in_room );
	act( "$n has created $p!", ch, obj, NULL, TO_ROOM );
    }
    send_to_char( "Ok.\r\n", ch );
	
	SET_BIT( obj->extra_flags, ITEM_GENERATED );
	
    return;
}



void do_purge( CHAR_DATA *ch, char *argument )
{
    char arg[MAX_INPUT_LENGTH];
    CHAR_DATA *victim;
    OBJ_DATA *obj;

    one_argument( argument, arg );

    if ( arg[0] == '\0' )
    {
	/* 'purge' */
	CHAR_DATA *vnext;
	OBJ_DATA  *obj_next;

	for ( victim = ch->in_room->people; victim != NULL; victim = vnext )
	{
	    vnext = victim->next_in_room;
	    if ( IS_NPC(victim) )
		extract_char( victim, TRUE );
	}

	for ( obj = ch->in_room->contents; obj != NULL; obj = obj_next )
	{
	    obj_next = obj->next_content;
	    extract_obj( obj );
	}

	act( "$n purges the room!", ch, NULL, NULL, TO_ROOM);
	send_to_char( "Ok.\r\n", ch );
	return;
    }

    if ( ( victim = get_char_world( ch, arg ) ) == NULL )
    {
	send_to_char( "They aren't here.\r\n", ch );
	return;
    }

    if ( !IS_NPC(victim) )
    {
	send_to_char( "Not on PC's.\r\n", ch );
	return;
    }

    act( "$n purges $N.", ch, NULL, victim, TO_NOTVICT );
    extract_char( victim, TRUE );
    return;
}



void do_advance( CHAR_DATA *ch, char *argument )
{
    char arg1[MAX_INPUT_LENGTH];
    char arg2[MAX_INPUT_LENGTH];
    CHAR_DATA *victim;
    int level;
    int iLevel;

    argument = one_argument( argument, arg1 );
    argument = one_argument( argument, arg2 );

    if ( arg1[0] == '\0' || arg2[0] == '\0' || !is_number( arg2 ) )
    {
	send_to_char( "Syntax: advance <char> <level>.\r\n", ch );
	return;
    }

    if ( ( victim = get_char_room( ch, arg1 ) ) == NULL )
    {
	send_to_char( "That player is not here.\r\n", ch);
	return;
    }

    if ( IS_NPC(victim) )
    {
	send_to_char( "Not on NPC's.\r\n", ch );
	return;
    }

    if ( ( level = atoi( arg2 ) ) < 1 || level > MAX_LEVEL )
    {
	printf_to_char( ch, "Level must be 1 to %d.\r\n", MAX_LEVEL );
	return;
    }

    if ( level > get_trust( ch ) )
    {
	send_to_char( "Limited to your trust level.\r\n", ch );
	return;
    }

    /*
     * Lower level:
     *   Reset to level 1.
     *   Then raise again.
     *   Currently, an imp can lower another imp.
     *   -- Swiftest
     */
	 
	 //fixme this should wipe skills
	 
    if ( level <= victim->level )
    {
	printf_to_char(ch, "Lowering %s's level to %d.\r\n", victim->name, level);
	victim->level    = 1;
	victim->exp      = 0;
	advance_level( victim );
    }
    else
    {
	printf_to_char(ch, "Raising %s's level to %d.\r\n", victim->name, level);
    }

    for ( iLevel = victim->level ; iLevel < level; iLevel++ )
    {
	victim->level += 1;
	victim->exp += get_exp_required(victim->level);
	advance_level( victim );
    }

    printf_to_char( victim, "You are now level %d.\r\n", victim->level );
    victim->trust = 0;
	
	if( victim->pcdata->deeds == 0 )
	{
		if( IS_SET( victim->act, PLR_COLOR ) )
		send_to_char( "{m", victim );	
			
		printf_to_char( victim, "The gods grant you three favors for free.\r\n");
		victim->pcdata->deeds += 3;
		
		if( IS_SET( victim->act, PLR_COLOR ) )
		send_to_char( "{x", victim );
	}
	
	sprintf( arg1, "%s all 0", victim->name );
	do_sset( ch, arg1 ); // this should work, and ptps/mtps are added by advance_level
    return;
}



void do_trust( CHAR_DATA *ch, char *argument )
{
    char arg1[MAX_INPUT_LENGTH];
    char arg2[MAX_INPUT_LENGTH];
    CHAR_DATA *victim;
    int level;

    argument = one_argument( argument, arg1 );
    argument = one_argument( argument, arg2 );

    if ( arg1[0] == '\0' || arg2[0] == '\0' || !is_number( arg2 ) )
    {
	send_to_char( "Syntax: trust <char> <level>.\r\n", ch );
	return;
    }

    if ( ( victim = get_char_room( ch, arg1 ) ) == NULL )
    {
	send_to_char( "That player is not here.\r\n", ch);
	return;
    }

    if ( ( level = atoi( arg2 ) ) < 0 || level > MAX_LEVEL )
    {
	printf_to_char( ch, "Level must be 0 (reset) or 1 to %d.\r\n", MAX_LEVEL );
	return;
    }

    if ( level > get_trust( ch ) )
    {
	send_to_char( "Limited to your trust.\r\n", ch );
	return;
    }

    victim->trust = level;
    return;
}



void do_restore( CHAR_DATA *ch, char *argument )
{
    char arg[MAX_INPUT_LENGTH];
    CHAR_DATA *victim;
	int iBody = 1;
	
    one_argument( argument, arg );
    if ( arg[0] == '\0' )
    {
	send_to_char( "Restore whom?\r\n", ch );
	return;
    }

    if ( ( victim = get_char_world( ch, arg ) ) == NULL )
    {
	send_to_char( "They aren't here.\r\n", ch );
	return;
    }

	while( iBody < MAX_BODY )
	{
		if( victim->body != 0 )
		{
		victim->bandage[iBody] = 0;
		victim->bleed[iBody] = 0;
		victim->body[iBody] = 0;
		victim->scars[iBody] = 0;
		}
		
	iBody++;
	}
	
	if( IS_SET( victim->affected_by, AFF_DEAD ) )
		REMOVE_BIT( victim->affected_by, AFF_DEAD );
	
    victim->hit  = victim->max_hit;
    victim->mana = victim->max_mana;
    victim->move = victim->max_move;
    victim->position = POS_STANDING;
	victim->wait = 0;
	victim->stun = 0;
	
	printf_to_char( victim, "You are healed!\r\n" );
    send_to_char( "Ok.\r\n", ch );
    return;
}



void do_freeze( CHAR_DATA *ch, char *argument )
{
    char arg[MAX_INPUT_LENGTH];
    CHAR_DATA *victim;

    one_argument( argument, arg );

    if ( arg[0] == '\0' )
    {
	send_to_char( "Freeze whom?\r\n", ch );
	return;
    }

    if ( ( victim = get_char_world( ch, arg ) ) == NULL )
    {
	send_to_char( "They aren't here.\r\n", ch );
	return;
    }

    if ( IS_NPC(victim) )
    {
	send_to_char( "Not on NPC's.\r\n", ch );
	return;
    }

    if ( get_trust( victim ) >= get_trust( ch ) )
    {
	send_to_char( "You failed.\r\n", ch );
	return;
    }

    if ( IS_SET(victim->act, PLR_FREEZE) )
    {
	REMOVE_BIT(victim->act, PLR_FREEZE);
	send_to_char( "You can play again.\r\n", victim );
	send_to_char( "FREEZE removed.\r\n", ch );
    }
    else
    {
	SET_BIT(victim->act, PLR_FREEZE);
	send_to_char( "You can't do ANYthing!\r\n", victim );
	send_to_char( "FREEZE set.\r\n", ch );
    }

    save_char_obj( victim );

    return;
}



void do_log( CHAR_DATA *ch, char *argument )
{
    char arg[MAX_INPUT_LENGTH];
    CHAR_DATA *victim;

    one_argument( argument, arg );

    if ( arg[0] == '\0' )
    {
	send_to_char( "Log whom?\r\n", ch );
	return;
    }

    if ( !str_cmp( arg, "all" ) )
    {
	if ( fLogAll )
	{
	    fLogAll = FALSE;
	    send_to_char( "Log ALL off.\r\n", ch );
	}
	else
	{
	    fLogAll = TRUE;
	    send_to_char( "Log ALL on.\r\n", ch );
	}
	return;
    }

    if ( ( victim = get_char_world( ch, arg ) ) == NULL )
    {
	send_to_char( "They aren't here.\r\n", ch );
	return;
    }

    if ( IS_NPC(victim) )
    {
	send_to_char( "Not on NPC's.\r\n", ch );
	return;
    }

    /*
     * No level check, gods can log anyone.
     */
    if ( IS_SET(victim->act, PLR_LOG) )
    {
	REMOVE_BIT(victim->act, PLR_LOG);
	send_to_char( "LOG removed.\r\n", ch );
    }
    else
    {
	SET_BIT(victim->act, PLR_LOG);
	send_to_char( "LOG set.\r\n", ch );
    }

    return;
}



void do_noemote( CHAR_DATA *ch, char *argument )
{
    char arg[MAX_INPUT_LENGTH];
    CHAR_DATA *victim;

    one_argument( argument, arg );

    if ( arg[0] == '\0' )
    {
	send_to_char( "Noemote whom?\r\n", ch );
	return;
    }

    if ( ( victim = get_char_world( ch, arg ) ) == NULL )
    {
	send_to_char( "They aren't here.\r\n", ch );
	return;
    }

    if ( IS_NPC(victim) )
    {
	send_to_char( "Not on NPC's.\r\n", ch );
	return;
    }

    if ( get_trust( victim ) >= get_trust( ch ) )
    {
	send_to_char( "You failed.\r\n", ch );
	return;
    }

    if ( IS_SET(victim->act, PLR_NO_EMOTE) )
    {
	REMOVE_BIT(victim->act, PLR_NO_EMOTE);
	send_to_char( "You can emote again.\r\n", victim );
	send_to_char( "NO_EMOTE removed.\r\n", ch );
    }
    else
    {
	SET_BIT(victim->act, PLR_NO_EMOTE);
	send_to_char( "You can't emote!\r\n", victim );
	send_to_char( "NO_EMOTE set.\r\n", ch );
    }

    return;
}



void do_notell( CHAR_DATA *ch, char *argument )
{
    char arg[MAX_INPUT_LENGTH];
    CHAR_DATA *victim;

    one_argument( argument, arg );

    if ( arg[0] == '\0' )
    {
	send_to_char( "Notell whom?", ch );
	return;
    }

    if ( ( victim = get_char_world( ch, arg ) ) == NULL )
    {
	send_to_char( "They aren't here.\r\n", ch );
	return;
    }

    if ( IS_NPC(victim) )
    {
	send_to_char( "Not on NPC's.\r\n", ch );
	return;
    }

    if ( get_trust( victim ) >= get_trust( ch ) )
    {
	send_to_char( "You failed.\r\n", ch );
	return;
    }

    if ( IS_SET(victim->act, PLR_NO_TELL) )
    {
	REMOVE_BIT(victim->act, PLR_NO_TELL);
	send_to_char( "You can tell again.\r\n", victim );
	send_to_char( "NO_TELL removed.\r\n", ch );
    }
    else
    {
	SET_BIT(victim->act, PLR_NO_TELL);
	send_to_char( "You can't tell!\r\n", victim );
	send_to_char( "NO_TELL set.\r\n", ch );
    }

    return;
}



void do_silence( CHAR_DATA *ch, char *argument )
{
    char arg[MAX_INPUT_LENGTH];
    CHAR_DATA *victim;

    one_argument( argument, arg );

    if ( arg[0] == '\0' )
    {
	send_to_char( "Silence whom?", ch );
	return;
    }

    if ( ( victim = get_char_world( ch, arg ) ) == NULL )
    {
	send_to_char( "They aren't here.\r\n", ch );
	return;
    }

    if ( IS_NPC(victim) )
    {
	send_to_char( "Not on NPC's.\r\n", ch );
	return;
    }

    if ( get_trust( victim ) >= get_trust( ch ) )
    {
	send_to_char( "You failed.\r\n", ch );
	return;
    }

    if ( IS_SET(victim->act, PLR_SILENCE) )
    {
	REMOVE_BIT(victim->act, PLR_SILENCE);
	send_to_char( "You can use channels again.\r\n", victim );
	send_to_char( "SILENCE removed.\r\n", ch );
    }
    else
    {
	SET_BIT(victim->act, PLR_SILENCE);
	send_to_char( "You can't use channels!\r\n", victim );
	send_to_char( "SILENCE set.\r\n", ch );
    }

    return;
}



void do_peace( CHAR_DATA *ch, char *argument )
{

    return;
}



BAN_DATA *		ban_free;
BAN_DATA *		ban_list;

void do_ban( CHAR_DATA *ch, char *argument )
{
    char buf[MAX_STRING_LENGTH];
    char arg[MAX_INPUT_LENGTH];
    BAN_DATA *pban;

    if ( IS_NPC(ch) )
	return;

    one_argument( argument, arg );

    if ( arg[0] == '\0' )
    {
	strcpy( buf, "Banned sites:\r\n" );
	for ( pban = ban_list; pban != NULL; pban = pban->next )
	{
	    strcat( buf, pban->name );
	    strcat( buf, "\r\n" );
	}
	send_to_char( buf, ch );
	return;
    }

    for ( pban = ban_list; pban != NULL; pban = pban->next )
    {
	if ( !str_cmp( arg, pban->name ) )
	{
	    send_to_char( "That site is already banned!\r\n", ch );
	    return;
	}
    }

    if ( ban_free == NULL )
    {
	pban		= alloc_perm( sizeof(*pban) );
    }
    else
    {
	pban		= ban_free;
	ban_free	= ban_free->next;
    }

    pban->name	= str_dup( arg );
    pban->next	= ban_list;
    ban_list	= pban;
    send_to_char( "Ok.\r\n", ch );
    return;
}



void do_allow( CHAR_DATA *ch, char *argument )
{
    char arg[MAX_INPUT_LENGTH];
    BAN_DATA *prev;
    BAN_DATA *curr;

    one_argument( argument, arg );

    if ( arg[0] == '\0' )
    {
	send_to_char( "Remove which site from the ban list?\r\n", ch );
	return;
    }

    prev = NULL;
    for ( curr = ban_list; curr != NULL; prev = curr, curr = curr->next )
    {
	if ( !str_cmp( arg, curr->name ) )
	{
	    if ( prev == NULL )
		ban_list   = ban_list->next;
	    else
		prev->next = curr->next;

	    free_string( curr->name );
	    curr->next	= ban_free;
	    ban_free	= curr;
	    send_to_char( "Ok.\r\n", ch );
	    return;
	}
    }

    send_to_char( "Site is not banned.\r\n", ch );
    return;
}



void do_wizlock( CHAR_DATA *ch, char *argument )
{
    extern bool wizlock;
    wizlock = !wizlock;

    if ( wizlock )
	send_to_char( "Game wizlocked.\r\n", ch );
    else
	send_to_char( "Game un-wizlocked.\r\n", ch );

    return;
}



void do_slookup( CHAR_DATA *ch, char *argument )
{
    char buf[MAX_STRING_LENGTH];
    char arg[MAX_INPUT_LENGTH];
    int sn;

    one_argument( argument, arg );
    if ( arg[0] == '\0' )
    {
	send_to_char( "Slookup what?\r\n", ch );
	return;
    }

    if ( !str_cmp( arg, "all" ) )
    {
	for ( sn = 0; sn < MAX_SKILL; sn++ )
	{
	    if ( spell_table[sn].name == NULL )
		break;
	    sprintf( buf, "Sn: %4d Slot: %4d spell: '%s'\r\n",
		sn, spell_table[sn].slot, spell_table[sn].name );
	    send_to_char( buf, ch );
	}
    }
    else
    {
	if ( ( sn = spell_lookup( arg ) ) < 0 )
	{
	    send_to_char( "No such spell.\r\n", ch );
	    return;
	}

	sprintf( buf, "Sn: %4d Slot: %4d Skill/spell: '%s'\r\n",
	    sn, spell_table[sn].slot, spell_table[sn].name );
	send_to_char( buf, ch );
    }

    return;
}



void do_sset( CHAR_DATA *ch, char *argument )
{
    char arg1 [MAX_INPUT_LENGTH];
    char arg2 [MAX_INPUT_LENGTH];
    char arg3 [MAX_INPUT_LENGTH];
    CHAR_DATA *victim;
    int value;
    int sn;
    bool fAll;

    argument = one_argument( argument, arg1 );
    argument = one_argument( argument, arg2 );
    argument = one_argument( argument, arg3 );

    if ( arg1[0] == '\0' || arg2[0] == '\0' || arg3[0] == '\0' )
    {
	send_to_char( "Syntax: sset <victim> <skill> <value>\r\n",	ch );
	send_to_char( "or:     sset <victim> all     <value>\r\n",	ch );
	send_to_char( "Skill being any skill or spell.\r\n",		ch );
	return;
    }

    if ( ( victim = get_char_world( ch, arg1 ) ) == NULL )
    {
	send_to_char( "They aren't here.\r\n", ch );
	return;
    }

    if ( IS_NPC(victim) )
    {
	send_to_char( "Not on NPC's.\r\n", ch );
	return;
    }

    fAll = !str_cmp( arg2, "all" );
    sn   = 0;
    if ( !fAll && ( sn = skill_lookup( arg2 ) ) < 0 )
    {
	send_to_char( "No such skill.\r\n", ch );
	return;
    }

    /*
     * Snarf the value.
     */
    if ( !is_number( arg3 ) )
    {
	send_to_char( "Value must be numeric.\r\n", ch );
	return;
    }

    value = atoi( arg3 );
    if ( value < 0 || value > 1000 )
    {
	send_to_char( "Value range is 0 to 1000.\r\n", ch );
	return;
    }

    if ( fAll )
    {
	for ( sn = 0; sn < MAX_SKILL; sn++ )
	{
	    if ( skill_table[sn].name != NULL )
		victim->pcdata->learned[sn]	= value;
	}
    }
    else
    {
	victim->pcdata->learned[sn] = value;
    }

    return;
}



void do_mset( CHAR_DATA *ch, char *argument )
{
    char arg1 [MAX_INPUT_LENGTH];
    char arg2 [MAX_INPUT_LENGTH];
    char arg3 [MAX_INPUT_LENGTH];
    CHAR_DATA *victim;
    int value;

    smash_tilde( argument );
    argument = one_argument( argument, arg1 );
    argument = one_argument( argument, arg2 );
    strcpy( arg3, argument );

    if ( arg1[0] == '\0' || arg2[0] == '\0' || arg3[0] == '\0' )
    {
	send_to_char( "Syntax: mset <victim> <field>  <value>\r\n",	ch );
	send_to_char( "or:     mset <victim> <string> <value>\r\n",	ch );
	send_to_char( "\r\n",						ch );
	send_to_char( "Field being one of:\r\n",			ch );
	send_to_char( "  str end dex spd wil pot jdg int wis cha\r\n",	ch );
	send_to_char( "  sex class level gold hp mana move\r\n",		ch );
	send_to_char( "  practice exp thirst drunk full hunt AS DS",			ch );
	send_to_char( "\r\n",						ch );
	send_to_char( "String being one of:\r\n",			ch );
	send_to_char( "  name short long description spec\r\n",	ch );
	return;
    }

    if ( ( victim = get_char_world( ch, arg1 ) ) == NULL )
    {
	send_to_char( "They aren't here.\r\n", ch );
	return;
    }

    /*
     * Snarf the value (which need not be numeric).
     */
    value = is_number( arg3 ) ? atoi( arg3 ) : -1;

    /*
     * Set something.
     */
    if ( !str_cmp( arg2, "str" ) )
    {
	if ( IS_NPC(victim) )
	{
	    send_to_char( "Not on NPC's.\r\n", ch );
	    return;
	}

	if ( value < 0 || value > 100 )
	{
	    send_to_char( "Strength range is 0 to 100.\r\n", ch );
	    return;
	}

	victim->pcdata->perm_Strength = value;
	return;
    }
	
	if ( !str_cmp( arg2, "AS" ) )
    {
	if ( value < 0 || value > 100 )
	{
	    send_to_char( "AS range is 0 to 100.\r\n", ch );
	    return;
	}

	victim->as = value;
	return;
    }
	
	if ( !str_cmp( arg2, "DS" ) )
    {
	if ( value < 0 || value > 100 )
	{
	    send_to_char( "AS range is 0 to 100.\r\n", ch );
	    return;
	}

	victim->ds = value;
	return;
    }
	
	    if ( !str_cmp( arg2, "end" ) )
    {
	if ( IS_NPC(victim) )
	{
	    send_to_char( "Not on NPC's.\r\n", ch );
	    return;
	}

	if ( value < 0 || value > 100 )
	{
	    send_to_char( "Endurance range is 0 to 100.\r\n", ch );
	    return;
	}

	victim->pcdata->perm_Endurance = value;
	return;
    }
	
	    if ( !str_cmp( arg2, "dex" ) )
    {
	if ( IS_NPC(victim) )
	{
	    send_to_char( "Not on NPC's.\r\n", ch );
	    return;
	}

	if ( value < 0 || value > 100 )
	{
	    send_to_char( "Dexterity range is 0 to 100.\r\n", ch );
	    return;
	}

	victim->pcdata->perm_Dexterity = value;
	return;
    }
	
	    if ( !str_cmp( arg2, "spd" ) )
    {
	if ( IS_NPC(victim) )
	{
	    send_to_char( "Not on NPC's.\r\n", ch );
	    return;
	}

	if ( value < 0 || value > 100 )
	{
	    send_to_char( "Speed range is 0 to 100.\r\n", ch );
	    return;
	}

	victim->pcdata->perm_Speed = value;
	return;
    }
	
	    if ( !str_cmp( arg2, "wil" ) )
    {
	if ( IS_NPC(victim) )
	{
	    send_to_char( "Not on NPC's.\r\n", ch );
	    return;
	}

	if ( value < 0 || value > 100 )
	{
	    send_to_char( "Willpower range is 0 to 100.\r\n", ch );
	    return;
	}

	victim->pcdata->perm_Willpower = value;
	return;
    }
	
	    if ( !str_cmp( arg2, "pot" ) )
    {
	if ( IS_NPC(victim) )
	{
	    send_to_char( "Not on NPC's.\r\n", ch );
	    return;
	}

	if ( value < 0 || value > 100 )
	{
	    send_to_char( "Potency range is 0 to 100.\r\n", ch );
	    return;
	}

	victim->pcdata->perm_Potency = value;
	return;
    }
	
	    if ( !str_cmp( arg2, "jdg" ) )
    {
	if ( IS_NPC(victim) )
	{
	    send_to_char( "Not on NPC's.\r\n", ch );
	    return;
	}

	if ( value < 0 || value > 100 )
	{
	    send_to_char( "Judgement range is 0 to 100.\r\n", ch );
	    return;
	}

	victim->pcdata->perm_Judgement = value;
	return;
    }
	
	    if ( !str_cmp( arg2, "int" ) )
    {
	if ( IS_NPC(victim) )
	{
	    send_to_char( "Not on NPC's.\r\n", ch );
	    return;
	}

	if ( value < 0 || value > 100 )
	{
	    send_to_char( "Intelligence range is 0 to 100.\r\n", ch );
	    return;
	}

	victim->pcdata->perm_Intelligence = value;
	return;
    }
	
	    if ( !str_cmp( arg2, "wis" ) )
    {
	if ( IS_NPC(victim) )
	{
	    send_to_char( "Not on NPC's.\r\n", ch );
	    return;
	}

	if ( value < 0 || value > 100 )
	{
	    send_to_char( "Wisdom range is 0 to 100.\r\n", ch );
	    return;
	}

	victim->pcdata->perm_Wisdom = value;
	return;
    }
	
	    if ( !str_cmp( arg2, "cha" ) )
    {
	if ( IS_NPC(victim) )
	{
	    send_to_char( "Not on NPC's.\r\n", ch );
	    return;
	}

	if ( value < 0 || value > 100 )
	{
	    send_to_char( "Charm range is 0 to 100.\r\n", ch );
	    return;
	}

	victim->pcdata->perm_Charm = value;
	return;
    }


    if ( !str_cmp( arg2, "sex" ) )
    {
	if ( value < 0 || value > 2 )
	{
	    send_to_char( "Sex range is 0 to 2.\r\n", ch );
	    return;
	}
	victim->sex = value;
	return;
    }

    if ( !str_cmp( arg2, "class" ) )
    {
	if ( value < 0 || value >= MAX_CLASS )
	{
	    char buf[MAX_STRING_LENGTH];

	    sprintf( buf, "Class range is 0 to %d.\n", MAX_CLASS-1 );
	    send_to_char( buf, ch );
	    return;
	}
	victim->class = value;
	return;
    }

    if ( !str_cmp( arg2, "level" ) )
    {
	if ( !IS_NPC(victim) )
	{
	    send_to_char( "Not on PC's.\r\n", ch );
	    return;
	}

	if ( value < 1 || value > MAX_LEVEL )
	{
		printf_to_char( ch, "Level must be 1 to %d.\r\n", MAX_LEVEL );
	    return;
	}
	
	victim->level = value;
	return;
    }

    if ( !str_cmp( arg2, "gold" ) )
    {
	victim->gold = value;
	return;
    }

    if ( !str_cmp( arg2, "hp" ) )
    {
	if ( value < 1 || value > 30000 )
	{
	    send_to_char( "Hp range is 1 to 30,000 hit points.\r\n", ch );
	    return;
	}
	victim->max_hit = value;
	return;
    }

    if ( !str_cmp( arg2, "mana" ) )
    {
	if ( value < 0 || value > 30000 )
	{
	    send_to_char( "Mana range is 0 to 30,000 mana points.\r\n", ch );
	    return;
	}
	victim->max_mana = value;
	return;
    }

    if ( !str_cmp( arg2, "move" ) )
    {
	if ( value < 0 || value > 30000 )
	{
	    send_to_char( "Move range is 0 to 30,000 move points.\r\n", ch );
	    return;
	}
	victim->max_move = value;
	return;
    }

    if ( !str_cmp( arg2, "practice" ) )
    {
	if ( value < 0 || value > 100 )
	{
	    send_to_char( "Practice range is 0 to 100 sessions.\r\n", ch );
	    return;
	}
	victim->practice = value;
	return;
    }

    if ( !str_cmp( arg2, "exp" ) )
    {
	victim->alignment = value;
	return;
    }

    if ( !str_cmp( arg2, "thirst" ) )
    {
	if ( IS_NPC(victim) )
	{
	    send_to_char( "Not on NPC's.\r\n", ch );
	    return;
	}

	if ( value < 0 || value > 100 )
	{
	    send_to_char( "Thirst range is 0 to 100.\r\n", ch );
	    return;
	}

	victim->pcdata->condition[COND_THIRST] = value;
	return;
    }

    if ( !str_cmp( arg2, "drunk" ) )
    {
	if ( IS_NPC(victim) )
	{
	    send_to_char( "Not on NPC's.\r\n", ch );
	    return;
	}

	if ( value < 0 || value > 100 )
	{
	    send_to_char( "Drunk range is 0 to 100.\r\n", ch );
	    return;
	}

	victim->pcdata->condition[COND_DRUNK] = value;
	return;
    }

    if ( !str_cmp( arg2, "full" ) )
    {
	if ( IS_NPC(victim) )
	{
	    send_to_char( "Not on NPC's.\r\n", ch );
	    return;
	}

	if ( value < 0 || value > 100 )
	{
	    send_to_char( "Full range is 0 to 100.\r\n", ch );
	    return;
	}

	victim->pcdata->condition[COND_FULL] = value;
	return;
    }

    if ( !str_cmp( arg2, "name" ) )
    {
	if ( !IS_NPC(victim) )
	{
	    send_to_char( "Not on PC's.\r\n", ch );
	    return;
	}

	free_string( victim->name );
	victim->name = str_dup( arg3 );
	return;
    }

    if ( !str_cmp( arg2, "short" ) )
    {
	free_string( victim->short_descr );
	victim->short_descr = str_dup( arg3 );
	return;
    }

    if ( !str_cmp( arg2, "long" ) )
    {
	free_string( victim->long_descr );
	victim->long_descr = str_dup( arg3 );
	return;
    }

    if ( !str_cmp( arg2, "spec" ) )
    {
	if ( !IS_NPC(victim) )
	{
	    send_to_char( "Not on PC's.\r\n", ch );
	    return;
	}

	if ( ( victim->spec_fun = spec_lookup( arg3 ) ) == 0 )
	{
	    send_to_char( "No such spec fun.\r\n", ch );
	    return;
	}

	return;
    }
	
	if (!str_cmp(arg2, "hunt"))
    {
        CHAR_DATA *hunted = 0;

        if ( !IS_NPC(victim) )
        {
            send_to_char( "Not on PC's.\n\r", ch );
            return;
        }

        if ( str_cmp( arg3, "." ) )
          if ( (hunted = get_char_world(victim, arg3)) == NULL )
            {
              send_to_char("Mob couldn't locate the victim to hunt.\n\r", ch);
              return;
            }

        victim->hunting = hunted;
        return;
    }

    /*
     * Generate usage message.
     */
    do_mset( ch, "" );
    return;
}



void do_oset( CHAR_DATA *ch, char *argument )
{
    char arg1 [MAX_INPUT_LENGTH];
    char arg2 [MAX_INPUT_LENGTH];
    char arg3 [MAX_INPUT_LENGTH];
    OBJ_DATA *obj;
    int value;

    smash_tilde( argument );
    argument = one_argument( argument, arg1 );
    argument = one_argument( argument, arg2 );
    strcpy( arg3, argument );

    if ( arg1[0] == '\0' || arg2[0] == '\0' || arg3[0] == '\0' )
    {
	send_to_char( "Syntax: oset <object> <field>  <value>\r\n",	ch );
	send_to_char( "or:     oset <object> <string> <value>\r\n",	ch );
	send_to_char( "\r\n",						ch );
	send_to_char( "Field being one of:\r\n",			ch );
	send_to_char( "  value0 value1 value2 value3\r\n",		ch );
	send_to_char( "  extra wear level weight cost timer\r\n",	ch );
	send_to_char( "\r\n",						ch );
	send_to_char( "String being one of:\r\n",			ch );
	send_to_char( "  name short long ed\r\n",			ch );
	return;
    }

    if ( ( obj = get_obj_world( ch, arg1 ) ) == NULL )
    {
	send_to_char( "Nothing like that in hell, earth, or heaven.\r\n", ch );
	return;
    }

    /*
     * Snarf the value (which need not be numeric).
     */
    value = atoi( arg3 );

    /*
     * Set something.
     */
    if ( !str_cmp( arg2, "value0" ) || !str_cmp( arg2, "v0" ) )
    {
	obj->value[0] = value;
	return;
    }

    if ( !str_cmp( arg2, "value1" ) || !str_cmp( arg2, "v1" ) )
    {
	obj->value[1] = value;
	return;
    }

    if ( !str_cmp( arg2, "value2" ) || !str_cmp( arg2, "v2" ) )
    {
	obj->value[2] = value;
	return;
    }

    if ( !str_cmp( arg2, "value3" ) || !str_cmp( arg2, "v3" ) )
    {
	obj->value[3] = value;
	return;
    }

    if ( !str_cmp( arg2, "extra" ) )
    {
	obj->extra_flags = value;
	return;
    }

    if ( !str_cmp( arg2, "wear" ) )
    {
	obj->wear_flags = value;
	return;
    }

    if ( !str_cmp( arg2, "level" ) )
    {
	obj->level = value;
	return;
    }

    if ( !str_cmp( arg2, "weight" ) )
    {
	obj->weight = value;
	return;
    }

    if ( !str_cmp( arg2, "cost" ) )
    {
	obj->cost = value;
	return;
    }

    if ( !str_cmp( arg2, "timer" ) )
    {
	obj->timer = value;
	return;
    }

    if ( !str_cmp( arg2, "name" ) )
    {
	free_string( obj->name );
	obj->name = str_dup( arg3 );
	return;
    }

    if ( !str_cmp( arg2, "short" ) )
    {
	free_string( obj->short_descr );
	obj->short_descr = str_dup( arg3 );
	return;
    }

    if ( !str_cmp( arg2, "long" ) )
    {
	free_string( obj->description );
	obj->description = str_dup( arg3 );
	return;
    }

    if ( !str_cmp( arg2, "ed" ) )
    {
	EXTRA_DESCR_DATA *ed;

	argument = one_argument( argument, arg3 );
	if ( argument == NULL )
	{
	    send_to_char( "Syntax: oset <object> ed <keyword> <string>\r\n",
		ch );
	    return;
	}

	if ( extra_descr_free == NULL )
	{
	    ed			= alloc_perm( sizeof(*ed) );
	}
	else
	{
	    ed			= extra_descr_free;
	    extra_descr_free	= ed;
	}

	ed->keyword		= str_dup( arg3     );
	ed->description		= str_dup( argument );
	ed->next		= obj->extra_descr;
	obj->extra_descr	= ed;
	return;
    }

    /*
     * Generate usage message.
     */
    do_oset( ch, "" );
    return;
}


void do_rset( CHAR_DATA *ch, char *argument )
{
    char arg2 [MAX_INPUT_LENGTH];
    char arg3 [MAX_INPUT_LENGTH];
    ROOM_INDEX_DATA *location;
    int value;

    if ( IS_NPC( ch ) )
	return;

    if ( !can_build( ch, ch->in_room->area ) )
    {
	send_to_char("You do not have authorization to build in this zone.\n\r", ch );
	return;
    }

    argument = one_argument( argument, arg2 );
    strcpy( arg3, argument );

    if ( arg2[0] == '\0' || arg3[0] == '\0' )
    {
	send_to_char( "Syntax: rset <field> value\n\r",	ch );
	send_to_char( "\n\r",						ch );
	send_to_char( "Field being one of:\n\r",			ch );
	send_to_char( "  flags sector\n\r",		ch );
	send_to_char( "  name\n\r",				ch );
	return;
    }

    location = ch->in_room;

    /*
     * Set something.
     */
    if ( !str_cmp( arg2, "flags" ) )
    {
    if ( !is_number( arg3 ) )
    {
	send_to_char( "Value must be numeric.\n\r", ch );
	return;
    }
    value = atoi( arg3 );

	location->room_flags	= value;
	return;
    }

    if ( !str_cmp( arg2, "sector" ) )
    {
    if ( !is_number( arg3 ) )
    {
	send_to_char( "Value must be numeric.\n\r", ch );
	return;
    }
    value = atoi( arg3 );

	location->sector_type	= value;
	return;
    }

    if ( !str_cmp( arg2, "name" ) )
    {
	free_string( location->name );
	location->name = str_dup( arg3 );
	return;
    }

    /*
     * Generate usage message.
     */
    do_rset( ch, "" );
    return;
}



void do_users( CHAR_DATA *ch, char *argument )
{
    char buf[MAX_STRING_LENGTH];
    char buf2[MAX_STRING_LENGTH];
    DESCRIPTOR_DATA *d;
    int count;

    count	= 0;
    buf[0]	= '\0';
    for ( d = descriptor_list; d != NULL; d = d->next )
    {
	if ( d->character != NULL && can_see( ch, d->character ) )
	{
	    count++;
	    sprintf( buf + strlen(buf), "[%3d %2d] %s@%s\r\n",
		d->descriptor,
		d->connected,
		d->original  ? d->original->name  :
		d->character ? d->character->name : "(none)",
		d->host
		);
	}
    }

    sprintf( buf2, "%d user%s\r\n", count, count == 1 ? "" : "s" );
    send_to_char( buf2, ch );
    send_to_char( buf, ch );
    return;
}



/*
 * Thanks to Grodyn for pointing out bugs in this function.
 */
void do_force( CHAR_DATA *ch, char *argument )
{
    char arg[MAX_INPUT_LENGTH];

    argument = one_argument( argument, arg );

    if ( arg[0] == '\0' || argument[0] == '\0' )
    {
	send_to_char( "Force whom to do what?\r\n", ch );
	return;
    }

    if ( !str_cmp( arg, "all" ) )
    {
	CHAR_DATA *vch;
	CHAR_DATA *vch_next;

	for ( vch = char_list; vch != NULL; vch = vch_next )
	{
	    vch_next = vch->next;

	    if ( !IS_NPC(vch) && get_trust( vch ) < get_trust( ch ) )
	    {
		act( "You are compelled by $n to: $t", ch, argument, vch, TO_VICT );
		interpret( vch, argument );
	    }
	}
    }
    else
    {
	CHAR_DATA *victim;

	if ( ( victim = get_char_world( ch, arg ) ) == NULL )
	{
	    send_to_char( "They aren't here.\r\n", ch );
	    return;
	}

	if ( victim == ch )
	{
	    send_to_char( "Aye aye, right away!\r\n", ch );
	    return;
	}

	if ( get_trust( victim ) >= get_trust( ch ) )
	{
	    send_to_char( "Do it yourself!\r\n", ch );
	    return;
	}

	act( "You are compelled by $n to: $t", ch, argument, victim, TO_VICT );
	interpret( victim, argument );
    }

    send_to_char( "Ok.\r\n", ch );
    return;
}



/*
 * New routines by Dionysos.
 */
void do_invis( CHAR_DATA *ch, char *argument )
{
    if ( IS_NPC(ch) )
	return;

    if ( IS_SET(ch->act, PLR_WIZINVIS) )
    {
	REMOVE_BIT(ch->act, PLR_WIZINVIS);
	act( "$n appears.", ch, NULL, NULL, TO_ROOM );
	send_to_char( "You appear.\r\n", ch );
    }
    else
    {
	act( "$n disappears.", ch, NULL, NULL, TO_ROOM );
	send_to_char( "You disappear.\r\n", ch );
	SET_BIT(ch->act, PLR_WIZINVIS);
    }

    return;
}



void do_holylight( CHAR_DATA *ch, char *argument )
{
    if ( IS_NPC(ch) )
	return;

    if ( IS_SET(ch->act, PLR_HOLYLIGHT) )
    {
	REMOVE_BIT(ch->act, PLR_HOLYLIGHT);
	send_to_char( "You see like a mortal.\r\n", ch );
    }
    else
    {
	SET_BIT(ch->act, PLR_HOLYLIGHT);
	send_to_char( "You see all now.\r\n", ch );
    }

    return;
}

//OBJ_DATA *create_random_box int level )

void do_randbox( CHAR_DATA *ch, char *arg )
{
    OBJ_DATA *obj;
    int i;

    if ( is_number( arg ) )
	i = atoi( arg );
	else
	i = ch->level;

    obj = create_random_box( i );

	if (obj != NULL)
		obj_to_char( obj, ch );
	
	if( i != 0 )
	obj->value[3] = i * 20;

	SET_BIT( obj->extra_flags, ITEM_GENERATED );
	
	//obj_to_room( obj, ch->in_room);

  return;
}


void do_randcontainer( CHAR_DATA *ch, char *arg )
{
    OBJ_DATA *obj;
    int i;

    if ( is_number( arg ) )
	i = atoi( arg );
	else
	i = ch->level;

    obj = create_random_container( i );

	if (obj != NULL)
		obj_to_char( obj, ch );
	
	//if( i != 0 )
	//obj->value[3] = i * 20;
	
	//obj_to_room( obj, ch->in_room);

  return;
}

//OBJ_DATA *create_random_armor( int level )

void do_randarmor( CHAR_DATA *ch, char *arg )
{
    OBJ_DATA *obj;
    int i;

    if ( is_number( arg ) )
	i = atoi( arg );

    obj = create_random_armor( i );

  if (obj != NULL)
  {
	if( number_range(0,9) == 7 )
	enchant_armor( obj, number_range(0,i/6) );

    obj_to_char( obj, ch );
	
	SET_BIT( obj->extra_flags, ITEM_GENERATED );
  }

  return;
}


void do_randshield( CHAR_DATA *ch, char *arg )
{
    OBJ_DATA *obj;
    int i;

    if ( is_number( arg ) )
	i = atoi( arg );

    obj = create_random_shield ( i );

  if (obj != NULL)
  {
	//if( number_range(0,9) == 7 )
	//enchant_armor( obj, number_range(0,i/6) );

    obj_to_char( obj, ch );
	
	SET_BIT( obj->extra_flags, ITEM_GENERATED );
  }
  
  return;
}

//OBJ_DATA *create_random_weapon( int level )

void do_randweapon( CHAR_DATA *ch, char *arg )
{
    OBJ_DATA *obj;
    int i;

    if ( is_number( arg ) )
		i = atoi( arg );

    obj = create_random_weapon( i );

    if ( obj == NULL )
		return;

	if( number_range(0,9) == 7 )
		enchant_weapon( obj, number_range(0,i/6) );

    obj_to_char( obj, ch );
	SET_BIT( obj->extra_flags, ITEM_GENERATED );
	
    return;
}


void do_randmagic( CHAR_DATA *ch, char *arg )
{
    OBJ_DATA *obj;

    obj = create_random_magic_item( number_range(0, MAX_MAGIC_ITEM-1) );

    if ( obj == NULL )
	return;

    obj_to_char( obj, ch );
	SET_BIT( obj->extra_flags, ITEM_GENERATED );
	
    return;
}


void do_randgem( CHAR_DATA *ch, char *arg )
{
    OBJ_DATA *obj;
	int i;

    if ( is_number( arg ) )
	i = atoi( arg );

    obj = create_random_gem( i );

    if ( obj == NULL )
	return;

    obj_to_char( obj, ch );
	SET_BIT( obj->extra_flags, ITEM_GENERATED );
	
    return;
}



void do_viewskills( CHAR_DATA *ch, char *argument )
{
    char arg[MAX_INPUT_LENGTH];
    CHAR_DATA *victim;
    int sn;
    int col;

    one_argument( argument, arg );

    if ( arg[0] == '\0' )
    {
	send_to_char( " Syntax:  viewskills <player>.\n\r", ch );
	return;
    }

    if ( (victim = get_char_world( ch, argument ) ) == NULL )
    {
	send_to_char("No such person in the game.\n\r", ch );
	return;
    }

    col    = 0;

	if ( !IS_NPC( victim ) )
	{
	printf_to_char( ch, "%s is level %d.\r\n\r\n", victim->name, victim->level );  
	  
    for ( sn = 0; sn < MAX_SKILL; sn++ )
    {
	if ( skill_table[sn].name == NULL )
	    break;
	if ( victim->pcdata->learned[sn] < 1 )
	    continue;

	printf_to_char( ch, "%21s  %3d  %3d  ", skill_table[sn].name, victim->pcdata->learned[sn], 
	skill_bonus(victim->pcdata->learned[sn]) );

	if ( ++col % 2 == 0 )
	send_to_char( "\n\r", ch );
	}
	}
	else
	{
	send_to_char("Mobile has null prototype.\n\r", ch );
	return;
	}

	if ( col % 2 != 0 )
	    send_to_char( "\n\r", ch );

    return;
}
#include <stdio.h>
#include <stdlib.h>
#include <strings.h>
#include <time.h>
#include "merc.h"


void do_balance( CHAR_DATA *ch, char *argument )
{
    char buf[MAX_STRING_LENGTH];

    if ( IS_NPC( ch ) )
	return;

    if ( ch->in_room == NULL || !IS_SET( ch->in_room->room_flags, ROOM_BANK ) )
    {
	send_to_char("There is no bank here.\n\r", ch );
	return;
    }

    if ( ch->pcdata->bank <= 0 )
    {
	send_to_char("You do not seem to have an account.\n\r", ch );
	return;
    }

    sprintf( buf, "You have %d coin%s in your account.\n\r", ch->pcdata->bank, ch->pcdata->bank == 1 ? "" : "s" );
    send_to_char( buf, ch );

    return;
}


void do_deposit( CHAR_DATA *ch, char *argument )
{
    char buf[MAX_STRING_LENGTH];
    int i;

    if ( IS_NPC( ch ) )
	return;

    if ( ch->in_room == NULL || !IS_SET( ch->in_room->room_flags, ROOM_BANK ) )
    {
	send_to_char("There is no bank here.\n\r", ch );
	return;
    }

    if ( !is_number( argument ) )
    {
	send_to_char("Deposit how many coins?\n\r", ch );
	return;
    }

    i = atoi( argument );

    if ( i <= 0 )
    {
	send_to_char("That's no way to save money.\n\r", ch );
	return;
    }

    if ( i > ch->gold )
    {
	send_to_char("You do not to seem to have that much on hand.\n\r", ch );
	return;
    }

    sprintf( buf, "You place %d coin%s into your account.\n\r", i, i == 1 ? "" : "s"  );
    send_to_char( buf, ch );
    ch->gold -= i;
    ch->pcdata->bank += i;

    return;
}

void do_withdraw( CHAR_DATA *ch, char *argument )
{
    char buf[MAX_STRING_LENGTH];
    int i;

    if ( IS_NPC( ch ) )
	return;

    if ( ch->in_room == NULL || !IS_SET( ch->in_room->room_flags, ROOM_BANK ) )
    {
	send_to_char("There is no bank here.\n\r", ch );
	return;
    }

    if ( !is_number( argument ) )
    {
	send_to_char("Withdraw how many coins?\n\r", ch );
	return;
    }

    i = atoi( argument );

    if ( i <= 0 )
    {
	send_to_char("You can't withraw a negative amount.\n\r", ch );
	return;
    }

    if ( i > ch->pcdata->bank )
    {
	send_to_char("You do not have that much in your account.\n\r", ch );
	return;
    }

    sprintf( buf, "You retrieve %d silver coin%s from your account.\n\r", i, i == 1 ? "" : "s"  );
    send_to_char( buf, ch );
    ch->gold += i;
    ch->pcdata->bank -= i;

    return;
}

/*
 * This file contains all of the OS-dependent stuff:
 *   startup, signals, BSD sockets for tcp/ip, i/o, timing.
 *
 * The data flow for input is:
 *    Game_loop ---> Read_from_descriptor ---> Read
 *    Game_loop ---> Read_from_buffer
 *
 * The data flow for output is:
 *    Game_loop ---> Process_Output ---> Write_to_descriptor -> Write
 *
 * The OS-dependent functions are Read_from_descriptor and Write_to_descriptor.
 * -- Furey  26 Jan 1993
 */

#include <stdlib.h>
#include <stdio.h>
#include <string.h>
#include <ctype.h>
#include <errno.h>
#include <unistd.h>
#include <sys/time.h>
#include <time.h>
#include <signal.h>
/*
 * Socket and TCP/IP stuff.
 */
#include <fcntl.h>
#include <netdb.h>
#include <netinet/in.h>
#include <sys/socket.h>
#include <arpa/telnet.h>
#include "merc.h"

const	char	echo_off_str	[] = { IAC, WILL, TELOPT_ECHO, '\0' };
const	char	echo_on_str	[] = { IAC, WONT, TELOPT_ECHO, '\0' };
const	char 	go_ahead_str	[] = { IAC, GA, '\0' };

/*
 * Global variables.
 */
DESCRIPTOR_DATA *   descriptor_free;	/* Free list for descriptors	*/
DESCRIPTOR_DATA *   descriptor_list;	/* All open descriptors		*/
DESCRIPTOR_DATA *   d_next;		/* Next descriptor in loop	*/
FILE *		    fpReserve;		/* Reserved file handle		*/
bool		    god;		/* All new chars are gods!	*/
bool		    merc_down;		/* Shutdown			*/
bool		    wizlock;		/* Game is wizlocked		*/
char		    str_boot_time[MAX_INPUT_LENGTH];
time_t		    current_time;	/* Time of this pulse		*/



/*
 * OS-dependent local functions.
 */
void	game_loop_unix		args( ( int control ) );
int	init_socket		args( ( int port ) );
void	new_descriptor		args( ( int control ) );
bool	read_from_descriptor	args( ( DESCRIPTOR_DATA *d ) );
bool	write_to_descriptor	args( ( int desc, char *txt, int length ) );

/*
 * Other local functions (OS-independent).
 */
bool	check_parse_name	args( ( char *name ) );
bool	check_reconnect		args( ( DESCRIPTOR_DATA *d, char *name,
				    bool fConn ) );
bool	check_playing		args( ( DESCRIPTOR_DATA *d, char *name ) );
int	main			args( ( int argc, char **argv ) );
void	nanny			args( ( DESCRIPTOR_DATA *d, char *argument ) );
bool	process_output		args( ( DESCRIPTOR_DATA *d, bool fPrompt ) );
void	read_from_buffer	args( ( DESCRIPTOR_DATA *d ) );
void	stop_idling		args( ( CHAR_DATA *ch ) );



int main( int argc, char **argv )
{
    struct timeval now_time;
    int port;
    int control;

    /*
     * Init time.
     */
    gettimeofday( &now_time, NULL );
    current_time = (time_t) now_time.tv_sec;
    strcpy( str_boot_time, ctime( &current_time ) );

    /*
     * Reserve one channel for our use.
     */
    if ( ( fpReserve = fopen( NULL_FILE, "r" ) ) == NULL )
    {
	perror( NULL_FILE );
	exit( 1 );
    }

    /*
     * Get the port number.
     */
    //port = 4000;
	port = 5501;
    if ( argc > 1 )
    {
	if ( !is_number( argv[1] ) )
	{
	    fprintf( stderr, "Usage: %s [port #]\n", argv[0] );
	    exit( 1 );
	}
	else if ( ( port = atoi( argv[1] ) ) <= 1024 && port != 23 )
	{
	    fprintf( stderr, "Port number must be 23 or above 1024.\n" );
	    exit( 1 );
	}
    }

    /*
     * Run the game.
     */
    control = init_socket( port );
    boot_db( );
    sprintf( log_buf, "Grinding on port %d.", port );
    log_string( log_buf );
    game_loop_unix( control );
    close( control );

    /*
     * That's all, folks.
     */
    log_string( "Normal termination of game." );
    exit( 0 );
    return 0;
}



int init_socket( int port )
{
    static struct sockaddr_in sa_zero;
    struct sockaddr_in sa;
    int x;
    int fd;

    if ( ( fd = socket( AF_INET, SOCK_STREAM, 0 ) ) < 0 )
    {
	perror( "Init_socket: socket" );
	exit( 1 );
    }

    if ( setsockopt( fd, SOL_SOCKET, SO_REUSEADDR,
    (char *) &x, sizeof(x) ) < 0 )
    {
	perror( "Init_socket: SO_REUSEADDR" );
	close( fd );
	exit( 1 );
    }

#if defined(SO_DONTLINGER) && !defined(SYSV)
    {
	struct	linger	ld;

	ld.l_onoff  = 1;
	ld.l_linger = 1000;

	if ( setsockopt( fd, SOL_SOCKET, SO_DONTLINGER,
	(char *) &ld, sizeof(ld) ) < 0 )
	{
	    perror( "Init_socket: SO_DONTLINGER" );
	    close( fd );
	    exit( 1 );
	}
    }
#endif

    sa		    = sa_zero;
    sa.sin_family   = AF_INET;
    sa.sin_port	    = htons( port );

    if ( bind( fd, (struct sockaddr *) &sa, sizeof(sa) ) < 0 )
    {
	perror( "Init_socket: bind" );
	close( fd );
	exit( 1 );
    }

    if ( listen( fd, 3 ) < 0 )
    {
	perror( "Init_socket: listen" );
	close( fd );
	exit( 1 );
    }

    return fd;
}



void game_loop_unix( int control )
{
    static struct timeval null_time;
    struct timeval last_time;

    signal( SIGPIPE, SIG_IGN );
	signal( SIGHUP, SIG_IGN); // necessary for valgrind debugging?
	
	
    gettimeofday( &last_time, NULL );
    current_time = (time_t) last_time.tv_sec;

    /* Main loop */
    while ( !merc_down )
    {
	fd_set in_set;
	fd_set out_set;
	fd_set exc_set;
	DESCRIPTOR_DATA *d;
	int maxdesc;

	/*
	 * Poll all active descriptors.
	 */
	FD_ZERO( &in_set  );
	FD_ZERO( &out_set );
	FD_ZERO( &exc_set );
	FD_SET( control, &in_set );
	maxdesc	= control;
	for ( d = descriptor_list; d; d = d->next )
	{
	    maxdesc = UMAX( maxdesc, d->descriptor );
	    FD_SET( d->descriptor, &in_set  );
	    FD_SET( d->descriptor, &out_set );
	    FD_SET( d->descriptor, &exc_set );
	}

	if ( select( maxdesc+1, &in_set, &out_set, &exc_set, &null_time ) < 0 )
	{
	    perror( "Game_loop: select: poll" );
	    exit( 1 );
	}

	/*
	 * New connection?
	 */
	if ( FD_ISSET( control, &in_set ) )
	    new_descriptor( control );

	/*
	 * Kick out the freaky folks.
	 */
	for ( d = descriptor_list; d != NULL; d = d_next )
	{
	    d_next = d->next;
	    if ( FD_ISSET( d->descriptor, &exc_set ) )
	    {
		FD_CLR( d->descriptor, &in_set  );
		FD_CLR( d->descriptor, &out_set );
		if ( d->character )
		    save_char_obj( d->character );
		d->outtop	= 0;
		close_socket( d );
	    }
	}

	/*
	 * Process input.
	 */
	for ( d = descriptor_list; d != NULL; d = d_next )
	{
	    d_next	= d->next;
	    d->fcommand	= FALSE;

	    if ( FD_ISSET( d->descriptor, &in_set ) )
	    {
		if ( d->character != NULL )
		    d->character->timer = 0;
		if ( !read_from_descriptor( d ) )
		{
		    FD_CLR( d->descriptor, &out_set );
		    if ( d->character != NULL )
			save_char_obj( d->character );
		    d->outtop	= 0;
		    close_socket( d );
		    continue;
		}
	    }

		/*
	    if ( d->character != NULL && d->character->wait > 0 )
	    {
		--d->character->wait;
	    }
		
		if ( d->character != NULL && d->character->stun > 0 )
	    {
		--d->character->stun;
	    }
		*/

	    read_from_buffer( d );
	    if ( d->incomm[0] != '\0' )
	    {
		d->fcommand	= TRUE;
		stop_idling( d->character );
		
		if( d->character != NULL )
		{	
		if( !IS_NPC( d->character ) && IS_SET( d->character->act, PLR_ECHO ) )
		send_to_char( d->incomm, d->character );
		
		if( !IS_NPC( d->character ) && IS_SET( d->character->act, PLR_RETURN ) )
		{
		send_to_char( "\r\n", d->character );
		}
		}
		
		if ( d->connected == CON_PLAYING )
		{
			interpret( d->character, d->incomm );
		}
		else
		{
		    nanny( d, d->incomm );	
		}			

		d->incomm[0]	= '\0';
	    }
	}


	/*
	 * Autonomous game motion.
	 */
	update_handler( );



	/*
	 * Output.
	 */
	for ( d = descriptor_list; d != NULL; d = d_next )
	{
	    d_next = d->next;

	    if ( ( d->fcommand || d->outtop > 0 )
	    &&   FD_ISSET(d->descriptor, &out_set) )
	    {
		if ( !process_output( d, TRUE ) )
		{
		    if ( d->character != NULL )
			save_char_obj( d->character );
		    d->outtop	= 0;
		    close_socket( d );
		}
	    }
	}



	/*
	 * Synchronize to a clock.
	 * Sleep( last_time + 1/PULSE_PER_SECOND - now ).
	 * Careful here of signed versus unsigned arithmetic.
	 */
	{
	    struct timeval now_time;
	    long secDelta;
	    long usecDelta;

	    gettimeofday( &now_time, NULL );
	    usecDelta	= ((int) last_time.tv_usec) - ((int) now_time.tv_usec)
			+ 1000000 / PULSE_PER_SECOND;
	    secDelta	= ((int) last_time.tv_sec ) - ((int) now_time.tv_sec );
	    while ( usecDelta < 0 )
	    {
		usecDelta += 1000000;
		secDelta  -= 1;
	    }

	    while ( usecDelta >= 1000000 )
	    {
		usecDelta -= 1000000;
		secDelta  += 1;
	    }

	    if ( secDelta > 0 || ( secDelta == 0 && usecDelta > 0 ) )
	    {
		struct timeval stall_time;

		stall_time.tv_usec = usecDelta;
		stall_time.tv_sec  = secDelta;
		if ( select( 0, NULL, NULL, NULL, &stall_time ) < 0 )
		{
		    perror( "Game_loop: select: stall" );
		    exit( 1 );
		}
	    }
	}

	gettimeofday( &last_time, NULL );
	current_time = (time_t) last_time.tv_sec;
    }

    return;
}



void new_descriptor( int control )
{
    static DESCRIPTOR_DATA d_zero;
    char buf[MAX_STRING_LENGTH];
    DESCRIPTOR_DATA *dnew;
    BAN_DATA *pban;
    struct sockaddr_in sock;
    struct hostent *from;
    int desc;
    int size;

    size = sizeof(sock);
    getsockname( control, (struct sockaddr *) &sock, &size );
    if ( ( desc = accept( control, (struct sockaddr *) &sock, &size) ) < 0 )
    {
	perror( "New_descriptor: accept" );
	return;
    }

#if !defined(FNDELAY)
#define FNDELAY O_NDELAY
#endif

    if ( fcntl( desc, F_SETFL, FNDELAY ) == -1 )
    {
	perror( "New_descriptor: fcntl: FNDELAY" );
	return;
    }

    /*
     * Cons a new descriptor.
     */
    if ( descriptor_free == NULL )
    {
	dnew		= alloc_perm( sizeof(*dnew) );
    }
    else
    {
	dnew		= descriptor_free;
	descriptor_free	= descriptor_free->next;
    }

    *dnew		= d_zero;
    dnew->descriptor	= desc;
    dnew->connected	= CON_GET_NAME;
    dnew->outsize	= 2000;
    dnew->outbuf	= alloc_mem( dnew->outsize );

    size = sizeof(sock);
    if ( getpeername( desc, (struct sockaddr *) &sock, &size ) < 0 )
    {
	perror( "New_descriptor: getpeername" );
	dnew->host = str_dup( "(unknown)" );
    }
    else
    {
	/*
	 * Would be nice to use inet_ntoa here but it takes a struct arg,
	 * which ain't very compatible between gcc and system libraries.
	 */
	int addr;

	addr = ntohl( sock.sin_addr.s_addr );
	sprintf( buf, "%d.%d.%d.%d",
	    ( addr >> 24 ) & 0xFF, ( addr >> 16 ) & 0xFF,
	    ( addr >>  8 ) & 0xFF, ( addr       ) & 0xFF
	    );
	sprintf( log_buf, "Sock.sinaddr:  %s", buf );
	log_string( log_buf );
	from = gethostbyaddr( (char *) &sock.sin_addr,
	    sizeof(sock.sin_addr), AF_INET );
	dnew->host = str_dup( from ? from->h_name : buf );
    }

//fixme no multiplaying
/*
    for ( dtest = descriptor_list; dtest != NULL; dtest = dtest->next )
    {
        if( !str_cmp( buf, dtest->host ) )
        {
            write_to_descriptor( desc,
        "Sorry,\n\rA connection from your site has already been established.\n\r", 0 );
            close( desc );
            free_mem( dnew->outbuf, dnew->outsize );
            dnew->next          = descriptor_free;
            descriptor_free     = dnew;
            return;
        }
    }
*/

    dnew->host = str_dup( buf );
	
    /*
     * Swiftest: I added the following to ban sites.  I don't
     * endorse banning of sites, but Copper has few descriptors now
     * and some people from certain sites keep abusing access by
     * using automated 'autodialers' and leaving connections hanging.
     *
     * Furey: added suffix check by request of Nickel of HiddenWorlds.
     */
    for ( pban = ban_list; pban != NULL; pban = pban->next )
    {
	if ( !str_suffix( pban->name, dnew->host ) )
	{
		//why tell them?
	    //write_to_descriptor( desc,
		//"Your site has been banned from this Mud.\n\r", 0 );
	    close( desc );
	    free_string( dnew->host );
	    free_mem( dnew->outbuf, dnew->outsize );
	    dnew->next		= descriptor_free;
	    descriptor_free	= dnew;
	    return;
	}
    }

    /*
     * Init descriptor data.
     */
    dnew->next			= descriptor_list;
    descriptor_list		= dnew;

    /*
     * Send the greeting.
     */
    {
	extern char * help_greeting;
	if ( help_greeting[0] == '.' )
	    write_to_buffer( dnew, help_greeting+1, 0 );
	else
	    write_to_buffer( dnew, help_greeting  , 0 );
    }

    return;
}





void close_socket( DESCRIPTOR_DATA *dclose )
{
    CHAR_DATA *ch;

    if ( dclose->outtop > 0 )
	process_output( dclose, FALSE );

    if ( dclose->snoop_by != NULL )
    {
	write_to_buffer( dclose->snoop_by,
	    "Your victim has left the game.\r\n", 0 );
    }

    {
	DESCRIPTOR_DATA *d;

	for ( d = descriptor_list; d != NULL; d = d->next )
	{
	    if ( d->snoop_by == dclose )
		d->snoop_by = NULL;
	}
    }

    if ( ( ch = dclose->character ) != NULL )
    {
	sprintf( log_buf, "Closing link to %s.", ch->name );
	log_string( log_buf );
	if ( dclose->connected == CON_PLAYING )
	{
		ch->desc = NULL;
		save_char_obj( ch );
		extract_char( ch, TRUE );
	    act( "$n has quit.", ch, NULL, NULL, TO_ROOM );
		
		//do_quit( ch, "" ); // auto quit as in GS
	}
	else
	{
	    free_char( dclose->original ? dclose->original : dclose->character );
	}
    }

    if ( d_next == dclose )
	d_next = d_next->next;

    if ( dclose == descriptor_list )
    {
	descriptor_list = descriptor_list->next;
    }
    else
    {
	DESCRIPTOR_DATA *d;

	for ( d = descriptor_list; d && d->next != dclose; d = d->next )
	    ;
	if ( d != NULL )
	    d->next = dclose->next;
	else
	    bug( "Close_socket: dclose not found.", 0 );
    }

    close( dclose->descriptor );
    free_string( dclose->host );
    dclose->next	= descriptor_free;
    descriptor_free	= dclose;
    return;
}



bool read_from_descriptor( DESCRIPTOR_DATA *d )
{
    int iStart;

    /* Hold horses if pending command already. */
    if ( d->incomm[0] != '\0' )
	return TRUE;

    /* Check for overflow. */
    iStart = strlen(d->inbuf);
    if ( iStart >= sizeof(d->inbuf) - 10 )
    {
	sprintf( log_buf, "%s input overflow!", d->host );
	log_string( log_buf );
	
	//fixme disco them.
	//write_to_descriptor( d->descriptor,
	 //   "\r\n*** PUT A LID ON IT!!! ***\r\n", 0 );
	return FALSE;
    }

    /* Snarf input. */
    for ( ; ; )
    {
	int nRead;

	nRead = read( d->descriptor, d->inbuf + iStart,
	    sizeof(d->inbuf) - 10 - iStart );
	if ( nRead > 0 )
	{
	    iStart += nRead;
	    if ( d->inbuf[iStart-1] == '\n' || d->inbuf[iStart-1] == '\r' )
		break;
	}
	else if ( nRead == 0 )
	{
	    log_string( "EOF encountered on read." );
	    return FALSE;
	}
	else if ( errno == EWOULDBLOCK )
	    break;
	else
	{
	    perror( "Read_from_descriptor" );
	    return FALSE;
	}
    }

    d->inbuf[iStart] = '\0';
    return TRUE;
}



/*
 * Transfer one line from input buffer to input line.
 */
void read_from_buffer( DESCRIPTOR_DATA *d )
{
    int i, j, k;

    /*
     * Hold horses if pending command already.
     */
    if ( d->incomm[0] != '\0' )
	return;

    /*
     * Look for at least one new line.
     */
    for ( i = 0; d->inbuf[i] != '\n' && d->inbuf[i] != '\r'; i++ )
    {
	if ( d->inbuf[i] == '\0' )
	    return;
    }

    /*
     * Canonical input processing.
     */
    for ( i = 0, k = 0; d->inbuf[i] != '\n' && d->inbuf[i] != '\r'; i++ )
    {
	if ( k >= MAX_INPUT_LENGTH - 2 )
	{
	    write_to_descriptor( d->descriptor, "Line too long.\r\n", 0 );

	    /* skip the rest of the line */
	    for ( ; d->inbuf[i] != '\0'; i++ )
	    {
		if ( d->inbuf[i] == '\n' || d->inbuf[i] == '\r' )
		    break;
	    }
	    d->inbuf[i]   = '\n';
	    d->inbuf[i+1] = '\0';
	    break;
	}

	if ( d->inbuf[i] == '\b' && k > 0 )
	    --k;
	else if ( isascii(d->inbuf[i]) && isprint((unsigned char) d->inbuf[i]) )
	    d->incomm[k++] = d->inbuf[i];
    }

    /*
     * Finish off the line.
     */
    if ( k == 0 )
	d->incomm[k++] = ' ';
    d->incomm[k] = '\0';

    /*
     * Deal with bozos with #repeat 1000 ...
     */
    if ( k > 1 || d->incomm[0] == '!' )
    {
    	if ( d->incomm[0] != '!' && strcmp( d->incomm, d->inlast ) )
	{
	    d->repeat = 0;
	}
	else
	{
	    if ( ++d->repeat >= 100 )
	    {
		sprintf( log_buf, "%s input spamming!", d->host );
		log_string( log_buf );
		strcpy( d->incomm, "quit" );
	    }
	}
    }

    /*
     * Do '!' substitution.
     */
    if ( d->incomm[0] == '!' )
	strcpy( d->incomm, d->inlast );
    else
	strcpy( d->inlast, d->incomm );

    /*
     * Shift the input buffer.
     */
    while ( d->inbuf[i] == '\n' || d->inbuf[i] == '\r' )
	i++;
    for ( j = 0; ( d->inbuf[j] = d->inbuf[i+j] ) != '\0'; j++ )
	;
    return;
}



/*
 * Low level output function.
 */
bool process_output( DESCRIPTOR_DATA *d, bool fPrompt )
{
    extern bool merc_down;

    /*
     * Bust a prompt.
     */
    if ( fPrompt && !merc_down && d->connected == CON_PLAYING )
    {
	CHAR_DATA *ch;

	ch = d->original ? d->original : d->character;
	if ( IS_SET(ch->act, PLR_BLANK) )
	    write_to_buffer( d, "\r\n", 2 );

	if ( IS_SET(ch->act, PLR_PROMPT) )
	{
	    char buf[40];
		bool bleed = FALSE;
		short b;
		
		for( b=0; b < MAX_BODY; b++ )
		{
			if( ch->bleed[b] > 0 && !IS_DEAD(ch) )
			{
				bleed = TRUE;
				break;
			}
		}
		
		//write_to_buffer( d, buf, 0 );
		
		/*
		   W ... Webbed *
   I ... Immobilized *
   i ... Invisible
   P ... Prone
   S ... Stunned
   s ... Sitting
   J ... Joined
   K ... Kneeling
   U ... Unconscious 
   H ... Hidden
   C ... Calmed *
   R ... In round-time delay
   ! ... Losing HPs due to bleed/disease/poison
DEAD ... Dead
*/

		if( IS_DEAD(ch) )
			sprintf(buf, "DEAD> ");
			
		else
			sprintf( buf, "%s%s%s%s%s%s%s%s%s%s> ", 
									ch->master != NULL ? "J" : "",
									IS_AFFECTED(ch, AFF_SLEEP) ? "U" : "",
									ch->stun > 0 ? "S" : "",
									ch->wait > 0 ? "R" : "",
									ch->position == POS_PRONE ? "P" : "",
									ch->position == POS_SITTING ? "s" : "",
									ch->position == POS_KNEELING ? "K" : "",
									bleed == TRUE ? "!" : "",
									IS_AFFECTED(ch, AFF_INVISIBLE) || IS_SET(ch->act, PLR_WIZINVIS) ? "I" : "",
									IS_AFFECTED(ch, AFF_HIDE) ? "H" : "");
		
		write_to_buffer( d, buf, 0 );
	}

	if ( IS_SET(ch->act, PLR_TELNET_GA) )
	    write_to_buffer( d, go_ahead_str, 0 );
    }

    /*
     * Short-circuit if nothing to write.
     */
    if ( d->outtop == 0 )
	return TRUE;

    /*
     * Snoop-o-rama.
     */
    if ( d->snoop_by != NULL )
    {
	write_to_buffer( d->snoop_by, "% ", 2 );
	write_to_buffer( d->snoop_by, d->outbuf, d->outtop );
    }

    /*
     * OS-dependent output.
     */
    if ( !write_to_descriptor( d->descriptor, d->outbuf, d->outtop ) )
    {
	d->outtop = 0;
	return FALSE;
    }
    else
    {
	d->outtop = 0;
	return TRUE;
    }
}



/*
 * Append onto an output buffer.
 */
void write_to_buffer( DESCRIPTOR_DATA *d, const char *txt, int length )
{
    /*
     * Find length in case caller didn't.
     */
    if ( length <= 0 )
	length = strlen(txt);

    /*
     * Initial \r\n if needed.
     */
    if ( d->outtop == 0 && !d->fcommand )
    {
	d->outbuf[0]	= '\r';
	d->outbuf[1]	= '\n';
	d->outtop	= 2;
    }

    /*
     * Expand the buffer as needed.
     */
    while ( d->outtop + length >= d->outsize )
    {
	char *outbuf;

	outbuf      = alloc_mem( 2 * d->outsize );
	strncpy( outbuf, d->outbuf, d->outtop );
	free_mem( d->outbuf, d->outsize );
	d->outbuf   = outbuf;
	d->outsize *= 2;
    }

    /*
     * Copy.
     */
    strncpy( d->outbuf + d->outtop, txt, length );
    d->outtop += length;
    return;
}



/*
 * Deal with sockets that haven't logged in yet.
 */

void nanny( DESCRIPTOR_DATA *d, char *argument )
{
    char buf[MAX_STRING_LENGTH];
    CHAR_DATA *ch;
    char *pwdnew;
    char *p;
    //int iClass;
	//int iRace;
    bool fOld;

/*
    if(argument[0] == 'C' && argument[1] == '>')
	{
		argument[0] == ' ';
		argument[1] == ' ';
	}
*/

    while ( isspace((unsigned char)*argument) )
	argument++;

    ch = d->character;

    switch ( d->connected )
    {
    case CON_GET_NAME:
	
		if ( argument[0] == '\0' )
		{
			close_socket( d );
			return;
		}
		
		//why do I need to fix this?
		if( !isalpha((unsigned char)argument[0]))
		strcpy(argument, "");
	
			//memmove(argument, argument+1, strlen(argument));

		argument[0] = UPPER(argument[0]);
		
			/* StormFront fixes are not worth it atm
		
		printf("argument: %s ", argument );
		
		if(argument[0] == '\<' && argument[1] == 'c' && argument[2] == '\>' && argument[3] == '\'' )
			memmove(argument, argument+4, strlen(argument));	
		
		if(argument[0] == '\<' && argument[1] == 'c' && argument[2] == '\>' )
			memmove(argument, argument+3, strlen(argument));	
		
		if(argument[0] == 'C' && argument[1] == '\>' && argument[2] == '\'' )
			memmove(argument, argument+3, strlen(argument));	
		
		if(argument[0] == 'C' && argument[1] == '\>' )
			memmove(argument, argument+2, strlen(argument));
		
		printf("-------------- fixed argument: %s\r\n", argument );	
	*/	
		
		if ( !check_parse_name( argument ) )
		{
			sprintf( log_buf, "There is something wrong with your name.\r\nEnter new character name: " );
			//printf("they put: %s\r\n", argument);
			write_to_buffer( d, log_buf, 0 );
			return;
		}

		fOld = load_char_obj( d, argument );
		ch   = d->character;

		if ( IS_SET(ch->act, PLR_DENY) )
		{
			sprintf( log_buf, "Denying access to %s@%s.", argument, d->host );
			log_string( log_buf );
			write_to_buffer( d, "You have been denied access.\r\n", 0 );
			close_socket( d );
			return;
		}

		if ( check_reconnect( d, argument, FALSE ) )
		{
			fOld = TRUE;
		}
		else
		{
			if ( wizlock && !IS_HERO(ch) )
			{
			write_to_buffer( d, "The game is running, but is currently closed for new connections.\r\n", 0 );
			close_socket( d );
			return;
			}
		}

		if ( fOld )
		{
			/* Old player */
			write_to_buffer( d, "Welcome back.  Enter your password: ", 0 );
			//write_to_buffer( d, echo_off_str, 0 );
			d->connected = CON_GET_OLD_PASSWORD;
			return;
		}
		else
		{
			/* New player */
			sprintf( buf, "Confirm your new character name is %s (Y/N)? ", argument );
			write_to_buffer( d, buf, 0 );
			d->connected = CON_CONFIRM_NEW_NAME;
			return;
		}
		
		break;

    case CON_GET_OLD_PASSWORD:
		
		write_to_buffer( d, "\r\n", 2 );
		
		/* StormFront fixes are not worth it atm
		
		printf("argument: %s ", argument );
		
		if(argument[0] == '\<' && argument[1] == 'c' && argument[2] == '\>' && argument[3] == '\'' )
			memmove(argument, argument+4, strlen(argument));	
		
		if(argument[0] == '\<' && argument[1] == 'c' && argument[2] == '\>' )
			memmove(argument, argument+3, strlen(argument));	
		
		if(argument[0] == 'C' && argument[1] == '\>' && argument[2] == '\'' )
			memmove(argument, argument+3, strlen(argument));	
		
		if(argument[0] == 'C' && argument[1] == '\>' )
			memmove(argument, argument+2, strlen(argument));
		
		printf("-------------- fixed argument: %s\r\n", argument );	
	*/	

		if ( strcmp( crypt( argument, ch->pcdata->pwd ), ch->pcdata->pwd ) )
		{
			write_to_buffer( d, "That was the wrong password.\r\n", 0 );
			close_socket( d );
			return;
		}

		write_to_buffer( d, echo_on_str, 0 );

	/*
		if ( check_reconnect( d, ch->name, TRUE ) )
			return;

		if ( check_playing( d, ch->name ) )
			return;
		
		*/
		
			check_playing( d, ch->name );

		if ( check_reconnect( d, ch->name, TRUE ) )
			return;
		
		// reload eq, anti-dupe measure.
		strcpy( argument, ch->name );
		free_char( ch );
		load_char_obj( d, argument );

		sprintf( log_buf, "%s@%s has connected.", ch->name, d->host );
		log_string( log_buf );

		write_to_buffer( d, "[Press Enter to rejoin the adventure.]\r\n", 0 );
		d->connected = CON_READ_MOTD;
		
		break;

    case CON_CONFIRM_NEW_NAME:
	
		/* StormFront fixes are not worth it atm
		
		printf("argument: %s ", argument );
		
		if(argument[0] == '\<' && argument[1] == 'c' && argument[2] == '\>' && argument[3] == '\'' )
			memmove(argument, argument+4, strlen(argument));	
		
		if(argument[0] == '\<' && argument[1] == 'c' && argument[2] == '\>' )
			memmove(argument, argument+3, strlen(argument));	
		
		if(argument[0] == 'C' && argument[1] == '\>' && argument[2] == '\'' )
			memmove(argument, argument+3, strlen(argument));	
		
		if(argument[0] == 'C' && argument[1] == '\>' )
			memmove(argument, argument+2, strlen(argument));
		
		printf("-------------- fixed argument: %s\r\n", argument );	
	*/	
		
		switch ( *argument )
		{
			case 'y': case 'Y':
				//send_to_char( VT100_ERASE_SCREEN, ch );
				//sprintf( buf, "Choose Password: %s", echo_off_str );
				
				//write_to_buffer( d, buf, 0 );
				
				write_to_buffer( d, "Enter new character password: ", 0 );
				d->connected = CON_GET_NEW_PASSWORD;
				break;

			case 'n': case 'N':
				write_to_buffer( d, "Enter new character name: ", 0 );
				free_char( d->character );
				d->character = NULL;
				d->connected = CON_GET_NAME;
				break;

			default:
			write_to_buffer( d, "Please just say (Y)es or (N)o: ", 0 );
			break;
		}
		
		break;

    case CON_GET_NEW_PASSWORD:
		
		//write_to_buffer( d, "\r\n", 2 );
		
		/* StormFront fixes are not worth it atm
		
		printf("argument: %s ", argument );
		
		if(argument[0] == '\<' && argument[1] == 'c' && argument[2] == '\>' && argument[3] == '\'' )
			memmove(argument, argument+4, strlen(argument));	
		
		if(argument[0] == '\<' && argument[1] == 'c' && argument[2] == '\>' )
			memmove(argument, argument+3, strlen(argument));	
		
		if(argument[0] == 'C' && argument[1] == '\>' && argument[2] == '\'' )
			memmove(argument, argument+3, strlen(argument));	
		
		if(argument[0] == 'C' && argument[1] == '\>' )
			memmove(argument, argument+2, strlen(argument));
		
		printf("-------------- fixed argument: %s\r\n", argument );	
	*/	

		if ( strlen(argument) < 5 )
		{
			write_to_buffer( d,
			"A new password must be a minium of 5 characters.\r\nEnter new character's password: ",
			0 );
			return;
		}

		pwdnew = crypt( argument, ch->name );
		for ( p = pwdnew; *p != '\0'; p++ )
		{
			if ( *p == '~' )
			{
			write_to_buffer( d,
				"There is something wrong with your password.\r\nEnter new character's password: ",
				0 );
			return;
			}
		}

		free_string( ch->pcdata->pwd );
		ch->pcdata->pwd	= str_dup( pwdnew );
		write_to_buffer( d, "Okay, but put your password in again: ", 0 );
		d->connected = CON_CONFIRM_NEW_PASSWORD;
		
		break;

    case CON_CONFIRM_NEW_PASSWORD:
	
		//write_to_buffer( d, "\r\n", 2 );
		
			/* StormFront fixes are not worth it atm
		
		printf("argument: %s ", argument );
		
		if(argument[0] == '\<' && argument[1] == 'c' && argument[2] == '\>' && argument[3] == '\'' )
			memmove(argument, argument+4, strlen(argument));	
		
		if(argument[0] == '\<' && argument[1] == 'c' && argument[2] == '\>' )
			memmove(argument, argument+3, strlen(argument));	
		
		if(argument[0] == 'C' && argument[1] == '\>' && argument[2] == '\'' )
			memmove(argument, argument+3, strlen(argument));	
		
		if(argument[0] == 'C' && argument[1] == '\>' )
			memmove(argument, argument+2, strlen(argument));
		
		printf("-------------- fixed argument: %s\r\n", argument );	
	*/	

		if ( strcmp( crypt( argument, ch->pcdata->pwd ), ch->pcdata->pwd ) )
		{
			write_to_buffer( d, "The two entered passwords for the new character do not match.\r\nEnter new character's password: ",
			0 );
			d->connected = CON_GET_NEW_PASSWORD;
			return;
		}

		//write_to_buffer( d, echo_on_str, 0 );

		write_to_buffer( d, "This new character is (M)ale or (F)emale: ", 0 );
		d->connected = CON_GET_NEW_SEX;
		
		break;

    case CON_GET_NEW_SEX:
	
		/* StormFront fixes are not worth it atm
		
		printf("argument: %s ", argument );
		
		if(argument[0] == '\<' && argument[1] == 'c' && argument[2] == '\>' && argument[3] == '\'' )
			memmove(argument, argument+4, strlen(argument));	
		
		if(argument[0] == '\<' && argument[1] == 'c' && argument[2] == '\>' )
			memmove(argument, argument+3, strlen(argument));	
		
		if(argument[0] == 'C' && argument[1] == '\>' && argument[2] == '\'' )
			memmove(argument, argument+3, strlen(argument));	
		
		if(argument[0] == 'C' && argument[1] == '\>' )
			memmove(argument, argument+2, strlen(argument));
		
		printf("-------------- fixed argument: %s\r\n", argument );	
		*/	
		
		switch ( argument[0] )
		{
		case 'm': case 'M': ch->sex = SEX_MALE;    break;
		case 'f': case 'F': ch->sex = SEX_FEMALE;  break;
		default:
		write_to_buffer( d, "This new character is (M)ale or (F)emale: ", 0 );
			return;
		}
		
		//fixme make numeric

		//ch->race = number_range(0,MAX_RACE);
		//ch->class = number_range(0,MAX_CLASS);
		
		sprintf( log_buf, "NEW PLAYER: %s the %s (from %s)", 
		ch->name, 
		race_table[ch->race].name,
		//class_table[ch->class].name,
		d->host );
		log_string( log_buf );
		//write_to_buffer( d, "[Press Enter to begin your adventure.]", 0 );
		do_help( ch, "motd" );
		d->connected = CON_READ_MOTD;	
		
		break;

    case CON_READ_MOTD:
	
		ch->next	= char_list;
		char_list	= ch;
		
		d->connected = CON_PLAYING;

		if ( ch->level == 0 )
		{

			//fixme default stats should be set here
			
			ch->pcdata->perm_Strength = 0;
			ch->pcdata->perm_Endurance = 0;
			ch->pcdata->perm_Dexterity = 0;
			ch->pcdata->perm_Speed = 0;
			ch->pcdata->perm_Willpower = 0;
			ch->pcdata->perm_Potency = 0;
			ch->pcdata->perm_Judgement = 0;
			ch->pcdata->perm_Intelligence = 0;
			ch->pcdata->perm_Wisdom = 0;
			ch->pcdata->perm_Charm = 0;
			
			//new defaults for "Web Client" / telnet
			//SET_BIT( ch->act, PLR_ECHO );
			//SET_BIT( ch->act, PLR_RETURN );
			
			//people like these
			SET_BIT( ch->act, PLR_COLOR );
			SET_BIT( ch->act, PLR_PROMPT );
			SET_BIT( ch->act, PLR_PARSE );
			SET_BIT( ch->act, PLR_COINS );
			//SET_BIT( ch->act, PLR_WRAP );
			//ch->pcdata->wrap = 77;
			ch->pcdata->wrap = 0;

			ch->level	= 0; //fixme level 0?
			ch->exp	= 0;
			ch->gold = 0;
			ch->practice = 0;
			ch->pcdata->MTPs = 0;
			ch->pcdata->PTPs = 0;
			
			ch->in_room = get_room_index( ROOM_VNUM_START );
			//write_to_buffer( d, VT100_ERASE_SCREEN, 0 );
			interpret(ch, "SELECT"); 
		
			//char_to_room( ch, get_room_index( ROOM_VNUM_START ) );
			
			/*
			char_to_room( ch, get_room_index( ROOM_VNUM_START ) );
			
			if( ch->in_room != NULL )
			{
				ch->in_room = get_room_index( ROOM_VNUM_START );
				printf("no in_room for %s\r\n", ch->name );
				
			}
			*/
			
			//printf_to_char(  ch, "You slowly awaken in a beautifully carved white marble temple.\r\n\r\n" );
			//printf_to_char(  ch, "You can barely remember a long journey, but your head feels heavy and full of fog.\r\n\r\n");

			//act( "$n pops into existence.", ch, NULL, NULL, TO_ROOM );

		}
		else if ( ch->in_room != NULL )
		{
			char_to_room( ch, ch->in_room );
			//write_to_buffer( d, VT100_ERASE_SCREEN, 0 );
		}
		else if ( IS_IMMORTAL(ch) )
		{
			char_to_room( ch, get_room_index( ROOM_VNUM_CHAT ) );
			//write_to_buffer( d, VT100_ERASE_SCREEN, 0 );
		}
		else
		{
			char_to_room( ch, get_room_index( ROOM_VNUM_TEMPLE ) );
			//write_to_buffer( d, VT100_ERASE_SCREEN, 0 );
		}
		
		//just to be safe here, we want people who connect to be able to save...
		if (!IS_SET(ch->act, PLR_CREATED ) )
			SET_BIT( ch->act, PLR_CREATED );
		
		act( "$n just arrived.", ch, NULL, NULL, TO_ROOM );
		
		
		do_look( ch, "auto" );
		
		break;
	
	default:
    
		bug( "Nanny: bad d->connected %d.", d->connected );
		close_socket( d );
		return;
		
    }

    return;
}




/*
 * Lowest level output function.
 * Write a block of text to the file descriptor.
 * If this gives errors on very long blocks (like 'ofind all'),
 *   try lowering the max block size.
 */
bool write_to_descriptor( int desc, char *txt, int length )
{
    int iStart;
    int nWrite;
    int nBlock;

    if ( length <= 0 )
	length = strlen(txt);

    for ( iStart = 0; iStart < length; iStart += nWrite )
    {
	nBlock = UMIN( length - iStart, 4096 );
	if ( ( nWrite = write( desc, txt + iStart, nBlock ) ) < 0 )
	    { perror( "Write_to_descriptor" ); return FALSE; }
    }

    return TRUE;
}




/*
 * Parse a name for acceptability.
 */
bool check_parse_name( char *name )
{
    /*
     * Reserved words.
     */
    if ( is_name( name, "all auto self someone god" ) )
	return FALSE;

    /*
     * Length restrictions.
     */
    if ( strlen(name) <  3 )
	return FALSE;

    if ( strlen(name) > 12 )
	return FALSE;

    /*
     * Alphanumerics only.
     * Lock out IllIll twits.
     */
    {
	char *pc;
	bool fIll;

	fIll = TRUE;
	for ( pc = name; *pc != '\0'; pc++ )
	{
	    if ( !isalpha((unsigned char)*pc) )
		return FALSE;
	    if ( LOWER(*pc) != 'i' && LOWER(*pc) != 'l' )
		fIll = FALSE;
	}

	if ( fIll )
	    return FALSE;
    }

    /*
     * Prevent players from naming themselves after mobs.
     */
    {
	extern MOB_INDEX_DATA *mob_index_hash[MAX_KEY_HASH];
	MOB_INDEX_DATA *pMobIndex;
	int iHash;

	for ( iHash = 0; iHash < MAX_KEY_HASH; iHash++ )
	{
	    for ( pMobIndex  = mob_index_hash[iHash];
		  pMobIndex != NULL;
		  pMobIndex  = pMobIndex->next )
	    {
		if ( is_name( name, pMobIndex->player_name ) )
		    return FALSE;
	    }
	}
    }

    return TRUE;
}


 

/*
 * Look for link-dead player to reconnect.
 */
bool check_reconnect( DESCRIPTOR_DATA *d, char *name, bool fConn )
{
    CHAR_DATA *ch;
    DESCRIPTOR_DATA *dclose;

    for ( ch = char_list; ch != NULL; ch = ch->next )
    {
	if ( !IS_NPC(ch)
	&& ( !fConn || ch->desc == NULL )
	&&   !str_cmp( d->character->name, ch->name ) )
	{
	    if ( fConn == FALSE )
	    {
		free_string( d->character->pcdata->pwd );
		d->character->pcdata->pwd = str_dup( ch->pcdata->pwd );
	    }
	    else
	    {
		d->character = ch;
		ch->desc	 = d;
		ch->timer	 = 0;
		send_to_char( "Reconnecting.\n\r", ch );
		act( "$n has reconnected.", ch, NULL, NULL, TO_ROOM );
		sprintf( log_buf, "%s@%s reconnected.", ch->name, d->host );
		log_string( log_buf );
		d->connected = CON_PLAYING;
	    }
	    return TRUE;
	}
    }

    for ( dclose = descriptor_list; dclose != NULL; dclose = dclose->next )
    {
	if ( dclose == d )
	    continue;

	if ( dclose->connected == CON_PLAYING )
	    continue;

	if ( d->character == NULL || dclose->character == NULL
	|| str_cmp( d->character->name, dclose->character->name ) )
	    continue;

	break;
    }

    if ( dclose == NULL )
	return FALSE;

/* free the original desc */
    free_char( dclose->character );

    if ( dclose == descriptor_list )
    {
	descriptor_list = descriptor_list->next;
    }
    else
    {
	DESCRIPTOR_DATA *d2;

	for ( d2 = descriptor_list; d2 && d2->next != dclose; d2 = d2->next )
	    ;
	if ( d2 != NULL )
	    d2->next = dclose->next;
	else
	    bug( "Close_socket: dclose not found.", 0 );
    }

    close( dclose->descriptor );
    free_string( dclose->host );
    dclose->next	= descriptor_free;
    descriptor_free	= dclose;

    return FALSE;
}




bool check_playing( DESCRIPTOR_DATA *d, char *name )
{
    DESCRIPTOR_DATA *dold;

    for ( dold = descriptor_list; dold; dold = dold->next )
    {
	if ( dold != d
	&&   dold->character != NULL
	&&   dold->connected != CON_GET_NAME
	&&   dold->connected != CON_GET_OLD_PASSWORD
	&&   !str_cmp( name, dold->original
	         ? dold->original->name : dold->character->name ) )
	{
	    write_to_buffer( dold, "You have reconnected in another session.  Goodbye!\n\r", 0 );
	    //write_to_buffer( d, "Reconnecting.\n\r", 0 );

	    close_socket( dold );

	    return TRUE;
	}
    }

    return FALSE;
}



void stop_idling( CHAR_DATA *ch )
{
    if ( ch == NULL
    ||   ch->desc == NULL
    ||   ch->desc->connected != CON_PLAYING
    ||   ch->was_in_room == NULL
    ||   ch->in_room != get_room_index( ROOM_VNUM_LIMBO ) )
	return;

    ch->timer = 0;
    char_from_room( ch );
    char_to_room( ch, ch->was_in_room );
    ch->was_in_room	= NULL;
    //act( "$n has returned from the void.", ch, NULL, NULL, TO_ROOM );
    return;
}



/*
 * Write to one char.
 */
void send_to_char_bw( const char *txt, CHAR_DATA *ch )
{

    if ( txt != NULL && ch->desc != NULL )
	{
	write_to_buffer( ch->desc, txt, strlen(txt) );
	}
	
    return;
}

int color( char type, CHAR_DATA *ch, char *string )
{
    char code[20];
    char *p = '\0';

    if ( IS_NPC(ch) )
	return ( 0 );

    switch ( type )
    {
	default:
	    sprintf( code, NORMAL );
	    break;
	
	case 'x':
	    sprintf( code, NORMAL );
	    break;
		
	case 'g':
	    sprintf( code, SAY );
	    break;
		
	case 'b':
	    sprintf( code, WHISPER );
	    break;
		
	case 'M':
	    sprintf( code, CHAT );
	    break;

	case 'm':
	    sprintf( code, MONSTERBOLD );
	    break;
		
	case 'r':
	    sprintf( code, ROOMTITLE );
	    break;

	case '{':
	    sprintf( code, "%c", '{' );
	    break;
    }

    p = code;
    while ( *p != '\0' )
    {
	*string	  = *p++;
	*++string = '\0';
    }

    return ( strlen( code ) );
}

void colorconvert( char *buf, const char *txt, CHAR_DATA *ch )
{
    const char *point;
    int skip = 0;

    if ( txt != NULL && ch->desc != NULL )
    {
	if ( !IS_NPC(ch) && IS_SET( ch->act, PLR_COLOR ) )
	{
	    for ( point = txt; *point; point++ )
	    {
		if ( *point == '{' )
		{
		    point++;
		    skip = color( *point, ch, buf );
		    while ( skip-- > 0 )
			++buf;
		    continue;
		}

		*buf   = *point;
		*++buf = '\0';
	    }

	    *buf = '\0';
	}
	else
	{
	    for ( point = txt; *point; point++ )
	    {
		if ( *point == '{' )
		{
		    point++;
		    continue;
		}

		*buf   = *point;
		*++buf = '\0';
	    }

	    *buf = '\0';
	}
    }

    return;
}

void send_to_char( const char *txt, CHAR_DATA *ch )
{
    const char *point;
    char *point2;
    char buf[MAX_STRING_LENGTH * 4];
    int skip = 0;

    buf[0] = '\0';
    point2 = buf;

    if ( txt != NULL && ch->desc != NULL )
    {
	if ( !IS_NPC(ch) && IS_SET( ch->act, PLR_COLOR ) )
	{
	    for ( point = txt; *point; point++ )
	    {
		if ( *point == '{' )
		{
		    point++;
		    skip = color( *point, ch, point2 );
		    while ( skip-- > 0 )
			++point2;
		    continue;
		}

		*point2   = *point;
		*++point2 = '\0';
	    }

	    *point2 = '\0';
		
		
		//if( !IS_NPC(ch) )
		write_to_buffer( ch->desc, buf, point2 - buf );
	}
	else
	{
	    for ( point = txt; *point; point++ )
	    {
		if ( *point == '{' )
		{
		    point++;
		    continue;
		}

		*point2   = *point;
		*++point2 = '\0';
	    }

	    *point2 = '\0';
		
		//if( !IS_NPC(ch) )
	    write_to_buffer( ch->desc, buf, point2 - buf );
	}
    }

    return;
}



void act( const char *format, CHAR_DATA *ch, const void *arg1, const void *arg2, int type )
{
    static char * const he_she	[] = { "it",  "he",  "she" };
    static char * const him_her	[] = { "it",  "him", "her" };
    static char * const his_her	[] = { "its", "his", "her" };

    char buf[MAX_STRING_LENGTH];
    char fname[MAX_INPUT_LENGTH];
    char buffer[MAX_STRING_LENGTH * 2];
    CHAR_DATA *to;
    CHAR_DATA *vch = (CHAR_DATA *) arg2;
    OBJ_DATA *obj1 = (OBJ_DATA  *) arg1;
    OBJ_DATA *obj2 = (OBJ_DATA  *) arg2;
    const char *str;
    const char *i;
    char *point;
    char *pbuff;
	//bool fColor=FALSE;

	//if( arg1 == NULL && arg2 == NULL )
		//return;

    /*
     * Discard null and zero-length messages.
     */
    if ( format == NULL || format[0] == '\0' )
	return;

    /*
     * Discard numm rooms and characters.
     */
    if ( ch == NULL || ch->in_room == NULL )
	return;

/*
	if( ch->in_room->vnum == ROOM_VNUM_START )
	return;
*/

    to = ch->in_room->people;
    if ( type == TO_VICT )
    {
	if ( vch == NULL )
	{
	    bug( "Act: null vch with TO_VICT.", 0 );
	    return;
	}

	if ( vch->in_room == NULL )
	    return;

	to = vch->in_room->people;
    }
    
    for ( ; to != NULL; to = to->next_in_room )
    {
	//if ( to->desc == NULL || to == NULL || !IS_AWAKE(to) )
	

if ( to->desc == NULL || to == NULL ||  to->in_room == NULL || !IS_AWAKE(to) )
	    continue;

	if ( type == TO_CHAR && to != ch )
	    continue;
	if ( type == TO_VICT && ( to != vch || to == ch ) )
	    continue;
	if ( type == TO_ROOM && to == ch )
	    continue;
	if ( type == TO_NOTVICT && (to == ch || to == vch) )
	    continue;

	point	= buf;
	str	= format;
	while ( *str != '\0' )
	{
	    if ( *str != '$' )
	    {
		*point++ = *str++;
		continue;
	    }

	    //fColor = TRUE;
	    ++str;
	    i = " <@@@> ";

	    if ( arg2 == NULL && *str >= 'A' && *str <= 'Z' )
	    {
		bug( "Act: missing arg2 for code %d.", *str );
		i = " <@@@> ";
	    }
	    else
	    {
		switch ( *str )
		{
		default:  bug( "Act: bad code %d.", *str );
			  i = " <@@@> ";				break;
		/* Thx alex for 't' idea */
		case 't': i = (char *) arg1;				break;
		case 'T': i = (char *) arg2;          			break;
		case 'c': i = CAP_PERS( ch,  to  );				break;
		case 'C': i = CAP_PERS( ch,  to  );				break;
		case 'n': i = PERS( ch,  to  );				break;
		case 'N': i = PERS( vch, to  );				break;
		case 'e': i = 	he_she  [URANGE(0, ch  ->sex, 2)];	break;
		case 'E': i = he_she  [URANGE(0, vch ->sex, 2)];	break;
		case 'm': i = him_her [URANGE(0, ch  ->sex, 2)];	break;
		case 'M': i = him_her [URANGE(0, vch ->sex, 2)];	break;
		case 's': i = his_her [URANGE(0, ch  ->sex, 2)];	break;
		case 'S': i = his_her [URANGE(0, vch ->sex, 2)];	break;
		case 'g': i = SAY; break; 		//color
		case 'b': i = WHISPER; break; //color
		case 'x': i = NORMAL; break;	//color

		case 'p':
		    i = can_see_obj( to, obj1 )
			    ? obj1->short_descr
			    : "something";
		    break;

		case 'P':
		    i = can_see_obj( to, obj2 )
			    ? obj2->short_descr
			    : "something";
		    break;
			
		//useful for obj socials.	
			
		case 'o':
		  i = can_see_obj( to, obj1 )
			    ? smash_article(obj1->short_descr)
			    : "something";
				break;
		
		case 'O':
		  i = can_see_obj( to, obj2 )
			    ? smash_article(obj2->short_descr)
			    : "something";
				break;
				
		case 'd':
		    if ( arg2 == NULL || ((char *) arg2)[0] == '\0' )
		    {
			i = "door";
		    }
		    else
		    {
			one_argument( (char *) arg2, fname );
			i = fname;
		    }
		    break;
		}
	    }
		
	    ++str;
	    while ( ( *point = *i ) != '\0' )
		++point, ++i;
	}

	*point++ = '\n';
	*point++ = '\r';
	*point   = '\0';
	
	{
	short first = 0;
	
	// this fixes three leading spaces in combat displays
	if( buf[0] == ' ' && buf[1] == ' ' && buf[2] == ' ' && buf[3] )
		first += 3;
	
	// this fixes two leading spaces in combat displays
	if( buf[0] == ' ' && buf[1] == ' ' && buf[2] != ' ' )
		first += 2;
	
	// this fixes color codes as leading character.
	if( buf[first] == '{' && buf[first+2] )
	first += 2;
	
	buf[first]   = UPPER(buf[first]);
	}
	
	pbuff	 = buffer;
	
	//if( fColor )
	colorconvert( pbuff, buf, to );
	
	write_to_buffer( to->desc, buffer, 0 );
    }

    return;
}

char *wrapstr( CHAR_DATA *ch, const char *str )
{
	static char strwrap[MAX_STRING_LENGTH];
	int i;
	int count = strlen(IS_NPC(ch) ? ch->short_descr : ch->name);

	for ( i = 0; i < strlen(str); i++ )
	{
	count++;
	if ( count > 66 && str[i] == ' ' )
	{
	strwrap[i] = '\n';
	strwrap[i+1] = '\r';
	count = 0;
	}
	else
	{
	strwrap[i] = str[i];
	}
	}
	strwrap[i] = '\0';
	return strwrap;
}


char *timestamp( char strtime[16] )
{
   int  count;
   char buf[MAX_STRING_LENGTH], tim[16];

   sprintf( buf, ctime( &current_time ) );

   for( count = 0; count < 16; count++ )
      tim[count] = buf[count+11];

   tim[5] = '\0';
   sprintf( strtime, tim );
   return str_dup(strtime);
}



#if defined(macintosh)
#include <types.h>
#else
#include <sys/types.h>
#endif
#include <stdio.h>
#include <time.h>
#include "merc.h"



/*
 * Class table.
 */
const	struct	class_type	class_table	[MAX_CLASS]	=
{
    {
	"Fighter",  
	SKILL_COMBAT,
	STAT_STRENGTH, STAT_ENDURANCE, 
	100, 400, 0
    },

    {
	"Thief",  
	SKILL_COMBAT,
	STAT_DEXTERITY, STAT_SPEED, 
	100, 400, 0
    },
	
	{
	"Priest",  
	SKILL_MAGIC,
	STAT_JUDGEMENT, STAT_WISDOM, 
	300, 100, 200
    },
	
    {
	"Mage",  
	SKILL_MAGIC,
	STAT_INTELLIGENCE, STAT_POTENCY, 
	900, 400, 500
    },
	
	{
	"Healer",  
	SKILL_MAGIC,
	STAT_ENDURANCE, STAT_WILLPOWER, 
	800, 100, 200
    },
	
	{
	"Sorcerer",  
	SKILL_MAGIC,
	STAT_POTENCY, STAT_WISDOM, 
	700, 400, 100
    } // Necromancer 
	
	//fixme add Bard, Ranger, Psionicist
	
};

const	struct	race_type	race_table	[MAX_RACE]	=
{
    {
	"Human", 80, "brown", "brown", "fair", 150, 90, 100, MANEUVER_AVERAGE, 6,
		 2,  2,  2,  2,  2,  2,  2,  2,  2,  2,
	BODY_HAS_R_EYE|BODY_HAS_L_EYE|BODY_HAS_HEAD|BODY_HAS_NECK|BODY_HAS_CHEST|BODY_HAS_BACK|BODY_HAS_R_ARM|BODY_HAS_R_HAND|BODY_HAS_L_ARM|BODY_HAS_L_HAND|BODY_HAS_ABDOMEN|BODY_HAS_R_LEG|BODY_HAS_R_FOOT|BODY_HAS_L_LEG|BODY_HAS_L_FOOT|BODY_HAS_CNS
    },
	
	{
	"Half-Giant", 100, "auburn", "hazel", "ruddy", 200, 120, 133, MANEUVER_AVERAGE, 7,
		 2,  2,  2,  2,  2,  2,  2,  2,  2,  2,
	BODY_HAS_R_EYE|BODY_HAS_L_EYE|BODY_HAS_HEAD|BODY_HAS_NECK|BODY_HAS_CHEST|BODY_HAS_BACK|BODY_HAS_R_ARM|BODY_HAS_R_HAND|BODY_HAS_L_ARM|BODY_HAS_L_HAND|BODY_HAS_ABDOMEN|BODY_HAS_R_LEG|BODY_HAS_R_FOOT|BODY_HAS_L_LEG|BODY_HAS_L_FOOT|BODY_HAS_CNS
    },
	
	{
	"Half-Elf", 400, "honey blonde", "blue", "pale", 135, 80, 92, MANEUVER_GOOD, 5,
		 2,  2,  2,  2,  2,  2,  2,  2,  2,  2,
	BODY_HAS_R_EYE|BODY_HAS_L_EYE|BODY_HAS_HEAD|BODY_HAS_NECK|BODY_HAS_CHEST|BODY_HAS_BACK|BODY_HAS_R_ARM|BODY_HAS_R_HAND|BODY_HAS_L_ARM|BODY_HAS_L_HAND|BODY_HAS_ABDOMEN|BODY_HAS_R_LEG|BODY_HAS_R_FOOT|BODY_HAS_L_LEG|BODY_HAS_L_FOOT|BODY_HAS_CNS
    },

    {
	"High Elf", 1500, "platinum blonde", "pale blue", "ivory", 130, 70, 78, MANEUVER_EXCELLENT, 5,
		 2,  2,  2,  2,  2,  2,  2,  2,  2,  2,
	BODY_HAS_R_EYE|BODY_HAS_L_EYE|BODY_HAS_HEAD|BODY_HAS_NECK|BODY_HAS_CHEST|BODY_HAS_BACK|BODY_HAS_R_ARM|BODY_HAS_R_HAND|BODY_HAS_L_ARM|BODY_HAS_L_HAND|BODY_HAS_ABDOMEN|BODY_HAS_R_LEG|BODY_HAS_R_FOOT|BODY_HAS_L_LEG|BODY_HAS_L_FOOT|BODY_HAS_CNS
    },
	
	{
	"Wood Elf", 1500, "maroon", "emerald green", "olive", 130, 70, 81, MANEUVER_GOOD, 5,
		 2,  2,  2,  2,  2,  2,  2,  2,  2,  2,
	BODY_HAS_R_EYE|BODY_HAS_L_EYE|BODY_HAS_HEAD|BODY_HAS_NECK|BODY_HAS_CHEST|BODY_HAS_BACK|BODY_HAS_R_ARM|BODY_HAS_R_HAND|BODY_HAS_L_ARM|BODY_HAS_L_HAND|BODY_HAS_ABDOMEN|BODY_HAS_R_LEG|BODY_HAS_R_FOOT|BODY_HAS_L_LEG|BODY_HAS_L_FOOT|BODY_HAS_CNS
    },
	
	{
	"Dark Elf", 1500, "white", "bright violet", "black", 120,  75, 84, MANEUVER_GOOD, 5,
		 2,  2,  2,  2,  2,  2,  2,  2,  2,  2,
	BODY_HAS_R_EYE|BODY_HAS_L_EYE|BODY_HAS_HEAD|BODY_HAS_NECK|BODY_HAS_CHEST|BODY_HAS_BACK|BODY_HAS_R_ARM|BODY_HAS_R_HAND|BODY_HAS_L_ARM|BODY_HAS_L_HAND|BODY_HAS_ABDOMEN|BODY_HAS_R_LEG|BODY_HAS_R_FOOT|BODY_HAS_L_LEG|BODY_HAS_L_FOOT|BODY_HAS_CNS
    },

    {
	"Dwarf", 180, "black", "brown", "ruddy", 140,  75, 80, MANEUVER_AVERAGE, 6,
		 2,  2,  2,  2,  2,  2,  2,  2,  2,  2,
	BODY_HAS_R_EYE|BODY_HAS_L_EYE|BODY_HAS_HEAD|BODY_HAS_NECK|BODY_HAS_CHEST|BODY_HAS_BACK|BODY_HAS_R_ARM|BODY_HAS_R_HAND|BODY_HAS_L_ARM|BODY_HAS_L_HAND|BODY_HAS_ABDOMEN|BODY_HAS_R_LEG|BODY_HAS_R_FOOT|BODY_HAS_L_LEG|BODY_HAS_L_FOOT|BODY_HAS_CNS
    },

    {
	"Halfling", 100, "light brown", "green", "fair", 100, 45, 50, MANEUVER_EXCELLENT, 4,
		 2,  2,  2,  2,  2,  2,  2,  2,  2,  2,
	BODY_HAS_R_EYE|BODY_HAS_L_EYE|BODY_HAS_HEAD|BODY_HAS_NECK|BODY_HAS_CHEST|BODY_HAS_BACK|BODY_HAS_R_ARM|BODY_HAS_R_HAND|BODY_HAS_L_ARM|BODY_HAS_L_HAND|BODY_HAS_ABDOMEN|BODY_HAS_R_LEG|BODY_HAS_R_FOOT|BODY_HAS_L_LEG|BODY_HAS_L_FOOT|BODY_HAS_CNS
    },
	
	{
	"Gnome", 200, "greasy black", "orange", "white", 100, 45, 60, MANEUVER_EXCELLENT, 4,
		 2,  2,  2,  2,  2,  2,  2,  2,  2,  2,
	BODY_HAS_R_EYE|BODY_HAS_L_EYE|BODY_HAS_HEAD|BODY_HAS_NECK|BODY_HAS_CHEST|BODY_HAS_BACK|BODY_HAS_R_ARM|BODY_HAS_R_HAND|BODY_HAS_L_ARM|BODY_HAS_L_HAND|BODY_HAS_ABDOMEN|BODY_HAS_R_LEG|BODY_HAS_R_FOOT|BODY_HAS_L_LEG|BODY_HAS_L_FOOT|BODY_HAS_CNS
    }
};


//fixme PC/NPC/Heroes table using old names


const	struct	skill_type	skill_table	[MAX_SKILL]	=
{
    {
		"Armor", SKILL_COMBAT, DIFFICULTY_NORMAL, 
		3, 0,
		STAT_STRENGTH, &gsn_armor_use
	},
	
	{
		"Shield", SKILL_COMBAT, DIFFICULTY_NORMAL, 
		3, 0,
		STAT_STRENGTH, &gsn_shield_use
	},
	
	{
		"Blade", SKILL_COMBAT, DIFFICULTY_NORMAL, 
		3, 1,
		STAT_STRENGTH, &gsn_edged_weapons
	},
	
	{
		"Blunt", SKILL_COMBAT, DIFFICULTY_NORMAL, 
		3, 1,
		STAT_STRENGTH, &gsn_blunt_weapons
	},
	
	{
		"Twohanders", SKILL_COMBAT, DIFFICULTY_NORMAL, 
		7, 2,
		STAT_STRENGTH, &gsn_two_handed_weapons
	},
	
	{
		"Polearms", SKILL_NOT_IMPLEMENTED, DIFFICULTY_NORMAL, 
		7, 2,
		STAT_STRENGTH, &gsn_polearm_weapons
	},
	
	{
		"Ranged", SKILL_NOT_IMPLEMENTED, DIFFICULTY_NORMAL, 
		3, 2,
		STAT_DEXTERITY, &gsn_ranged_weapons
	},
	
	{
		"Thrown", SKILL_NOT_IMPLEMENTED, DIFFICULTY_NORMAL, 
		3, 2,
		STAT_DEXTERITY, &gsn_thrown_weapons
	},
	
	{
		"Brawling", SKILL_COMBAT, DIFFICULTY_NORMAL, 
		2, 1,
		STAT_STRENGTH, &gsn_brawling
	},
		
	{
		"Ambush", SKILL_COMBAT, DIFFICULTY_NORMAL, 
		4, 2,
		STAT_DEXTERITY, &gsn_ambush
	},
		
	{
		"Dual Wield", SKILL_NOT_IMPLEMENTED, DIFFICULTY_NORMAL, 
		4, 2,
		STAT_DEXTERITY, &gsn_two_weapon_combat
	},
	
	{
		"Maneuevers", SKILL_COMBAT, DIFFICULTY_NORMAL, 
		5, 2,
		STAT_SPEED, &gsn_combat_maneuvers
	},
	
	{
		"Multi-Opps", SKILL_NOT_IMPLEMENTED, DIFFICULTY_NORMAL, 
		6, 3,
		STAT_SPEED, &gsn_multi_opponent_combat
	},
		
	{
		"Healthiness", SKILL_COMBAT, DIFFICULTY_NORMAL, 
		3, 0,
		STAT_ENDURANCE, &gsn_physical_fitness
	},
		
	{
		"Dodge", SKILL_COMBAT, DIFFICULTY_NORMAL, 
		3, 3,
		STAT_SPEED, &gsn_dodging
	},
		
	{
		"Scrolls", SKILL_MAGIC, DIFFICULTY_EASY, 
		0, 2,
		STAT_INTELLIGENCE, &gsn_arcane_symbols
	},
	
	{
		"Magic Items", SKILL_NOT_IMPLEMENTED, DIFFICULTY_EASY, 
		0, 2,
		STAT_INTELLIGENCE, &gsn_magic_item_use
	},
		
	{
		"Aimed Spells", SKILL_MAGIC, DIFFICULTY_NORMAL, 
		4, 3,
		STAT_DEXTERITY, &gsn_spell_aiming
	},
		
	{
		"Harness Mana", SKILL_MAGIC, DIFFICULTY_NORMAL, 
		0, 6,
		STAT_POTENCY, &gsn_harness_power
	},
		
	{
		"Black Mana Control", SKILL_MAGIC, DIFFICULTY_NORMAL, 
		0, 6,
		STAT_POTENCY, &gsn_elemental_mana_control
	},
	
	/*
	{
		"Mentalism", SKILL_MAGIC, DIFFICULTY_NORMAL, 
		0, 3,
		STAT_JUDGEMENT, &gsn_mental_mana_control
	},
	*/
	
		
	{
		"White Mana Control", SKILL_MAGIC, DIFFICULTY_NORMAL, 
		0, 6,
		STAT_WISDOM, &gsn_spirit_mana_control
	},
		
	{
		"Spell Research", SKILL_MAGIC, DIFFICULTY_NORMAL, 
		0, 20,
		STAT_INTELLIGENCE, &gsn_spell_research
	},
		
	{
		"Black Sorcery", SKILL_MAGIC, DIFFICULTY_NORMAL, 
		0, 6,
		STAT_POTENCY, &gsn_elemental_lore
	},
	
	{
		"White Wizardry", SKILL_MAGIC, DIFFICULTY_NORMAL, 
		0, 6,
		STAT_WISDOM, &gsn_spiritual_lore
	},
	
/*	
	{
		"Sorcery Lore", SKILL_NOT_IMPLEMENTED, DIFFICULTY_NORMAL, 
		0, 3,
		STAT_POTENCY, &gsn_sorcerous_lore
	},
		
	{
		"Mental Lore", SKILL_NOT_IMPLEMENTED, DIFFICULTY_NORMAL, 
		0, 3,
		STAT_JUDGEMENT, &gsn_mental_lore
	},
*/
		
	{
		"Survival", SKILL_GENERAL, DIFFICULTY_NORMAL, 
		2, 2,
		STAT_ENDURANCE, &gsn_survival
	},
	
	{
		"Disarming", SKILL_STEALTH, DIFFICULTY_NORMAL, 
		2, 2,
		STAT_DEXTERITY, &gsn_disarming_traps
	},
		
	{
		"Picking", SKILL_STEALTH, DIFFICULTY_NORMAL, 
		2, 2,
		STAT_DEXTERITY, &gsn_picking_locks
	},
		
	{
		"Hiding", SKILL_STEALTH, DIFFICULTY_NORMAL, 
		3, 3,
		STAT_DEXTERITY, &gsn_stalking_and_hiding
	},
		
	{
		"Perception", SKILL_STEALTH, DIFFICULTY_NORMAL, 
		0, 3,
		STAT_JUDGEMENT, &gsn_perception
	},
	
	{
		"Climbing", SKILL_NOT_IMPLEMENTED, DIFFICULTY_NORMAL, 
		3, 0,
		STAT_ENDURANCE, &gsn_climbing
	},
		
	{
		"Swimming", SKILL_NOT_IMPLEMENTED, DIFFICULTY_NORMAL, 
		3, 0,
		STAT_ENDURANCE, &gsn_swimming
	},
		
	{
		"First Aid", SKILL_GENERAL, DIFFICULTY_NORMAL, 
		2, 2,
		STAT_DEXTERITY, &gsn_first_aid
	},
		
	{
		"Trading", SKILL_NOT_IMPLEMENTED, DIFFICULTY_NORMAL, 
		0, 2,
		STAT_CHARM, &gsn_trading
	},
	
	{
		"Stealing", SKILL_STEALTH, DIFFICULTY_NORMAL, 
		2, 2,
		STAT_DEXTERITY, &gsn_pickpocketing
	}
};

/*
 * The skill and spell table.
 * Slot numbers must never be changed as they appear in #OBJECTS sections.
 */
#define SLOT(n)	n


//fixme add spells Calm, Manna Bread
const	struct 	spell_type 	spell_table[MAX_SPELL] =
{
	{ CLASS_MAGE,
	999, "reserved",		110,
	spell_null,			TAR_IGNORE,		POS_STANDING,
	NULL,			
	SLOT( 0),	 0,	 0,
	"",
	""
    },
	

	{ CLASS_PRIEST,
	101, "White Ward I",		1,
	spell_spirit_warding_i,	TAR_CHAR_DEFENSIVE,	POS_STANDING,
	&gsn_spirit_warding_i,			SLOT(101),	35,	12,
	"A light green glow fades from you.",
		"A light green glow fades from $n."
    },

	{ CLASS_HEALER,
	801, "Restore Health I",		1,
	spell_restore_health_i,	TAR_CHAR_HEALING,	POS_STANDING,
	&gsn_restore_health_i,			SLOT(801),	35,	12, NULL, NULL
    },
	
	{ CLASS_PRIEST,
	301, "Hold Undead",			1,
	spell_hold_undead,		TAR_CHAR_OFFENSIVE,	POS_STANDING,
	&gsn_hold_undead,			SLOT(301),	35,	12,
	"",
	""
    },
	
	
	{ CLASS_PRIEST,
	202, "Spirit Boost I",		2,
	spell_spirit_shield,	TAR_CHAR_DEFENSIVE,	POS_STANDING,
	&gsn_spirit_shield,			SLOT(202),	35,	12,
	"The faint effervescence fades from around you.",
	"The faint effervescence fades from around $n."
    },
	
	{ CLASS_PRIEST,
	102, "Spirit Barrier",		2,
	spell_spirit_barrier,	TAR_CHAR_DEFENSIVE,	POS_STANDING,
	&gsn_spirit_barrier,			SLOT(102),	35,	12,
	"The air stops churning around you.",
	"The air stops churning around $n."
    },
	
	{ CLASS_PRIEST,
	303, "Destroy Undead",			3,
	spell_destroy_undead,		TAR_CHAR_OFFENSIVE,	POS_STANDING,
	&gsn_destroy_undead,			SLOT(303),	35,	12,
	"",
	""
    },
	
	{ CLASS_PRIEST,
	103, "Spirit Boost II",		3,
	spell_spirit_defense,	TAR_CHAR_DEFENSIVE,	POS_STANDING,
	&gsn_spirit_defense,			SLOT(103),	35,	12,
	"You feel slightly less powerful.",
	"$n looks slightly less powerful."
    },
	
	{ CLASS_HEALER,
	803, "Heal Minor Wounds",		3,
	spell_heal_i,	TAR_CHAR_HEALING,	POS_STANDING,
	&gsn_heal_i,			SLOT(803),	35,	12, NULL, NULL
    },
	
	{ CLASS_PRIEST,
	304, "Bless Weapon",		4,
	spell_bless_weapon,		TAR_OBJ_INV,	POS_STANDING,
	NULL,			SLOT(304),	15,	12,
	"",
	""
    },
	
	{ CLASS_HEALER,
	805, "Cure Light Scars",		5,
	spell_heal_scars_i,	TAR_CHAR_HEALING,	POS_STANDING,
	&gsn_heal_scars_i,		SLOT(805),	35,	12, NULL, NULL
    },
	
	{ CLASS_PRIEST,
	306, "Holy Water",		6,
	spell_holy_bolt,	TAR_CHAR_OFFENSIVE,	POS_STANDING,
	NULL,			SLOT(306),	35,	12,
	"",
	""
    },
	
	{ CLASS_PRIEST,
	107, "White Ward II",		7,
	spell_spirit_warding_ii,	TAR_CHAR_DEFENSIVE,	POS_STANDING,
	&gsn_spirit_warding_ii,			SLOT(107),	35,	12,
	"A deep green glow fades from you.", 
	"A deep green glow fades from $n."
    },
		
	{ CLASS_HEALER,
	807, "Heal Moderate Wounds",		7,
	spell_heal_ii,	TAR_CHAR_HEALING,	POS_STANDING,
	&gsn_heal_ii,			SLOT(807),	35,	12, NULL, NULL
    },
	
	{ CLASS_PRIEST,
	108, "Unstun",		8,
	spell_unstun,	TAR_CHAR_DEFENSIVE,	POS_STANDING,
	NULL,			SLOT(108),	35,	12,
	"",
	""
    },
	
	{ CLASS_HEALER,
	809, "Cure Moderate Scars",		9,
	spell_heal_scars_ii,	TAR_CHAR_HEALING,	POS_STANDING,
	&gsn_heal_scars_ii,			SLOT(807),	35,	12, NULL, NULL
    },
	
	{ CLASS_HEALER,
	810, "Restore Health II",		10,
	spell_restore_health_ii,	TAR_CHAR_HEALING,	POS_STANDING,
	&gsn_restore_health_ii,			SLOT(810),	35,	12, NULL, NULL
    },
	
	{ CLASS_PRIEST,
	312, "Harm",			12,
	spell_harm,		TAR_CHAR_OFFENSIVE,	POS_STANDING,
	NULL,			SLOT(311),	35,	12,
	"",
	""
    },
	
	{ CLASS_HEALER,
	814, "Heal Critical Wounds",		14,
	spell_heal_iii,	TAR_CHAR_HEALING,	POS_STANDING,
	&gsn_heal_iii,			SLOT(814),	35,	12, NULL, NULL
    },
	
	{ CLASS_HEALER,
	816, "Cure Heavy Scars",		16,
	spell_heal_scars_iii,	TAR_CHAR_HEALING,	POS_STANDING,
	&gsn_heal_scars_iii,			SLOT(816),	35,	12, NULL, NULL
    },

	{ CLASS_PRIEST,
	318, "Resurrect Dead",		18,
	spell_resurrect,	TAR_CHAR_HEALING,	POS_STANDING,
	&gsn_resurrect,			SLOT(318),	35,	12, NULL, NULL
    },
	
	{ CLASS_PRIEST,
	219, "Spell Shield",		19,
	spell_spell_shield,	TAR_CHAR_DEFENSIVE,	POS_STANDING,
	&gsn_spell_shield,			SLOT(219),	35,	12,
	"The opalescent shield fades from around you.",
	"The opalescent shield fades from around $n."
    },
			
	{ CLASS_PRIEST,
	120, "Spirit Boost III",	20,
	spell_lesser_shroud,	TAR_CHAR_DEFENSIVE,	POS_STANDING,
	&gsn_lesser_shroud,			SLOT(120),	35,	12,
	"You feel much less powerful.",
	"$n looks much less powerful."
    },
	
	{ CLASS_PRIEST,
	225, "Word of Transfer",		25,
	spell_transference,	TAR_CHAR_DEFENSIVE,	POS_STANDING,
	&gsn_transference,			SLOT(225),	35,	12, NULL, NULL
    },
	
	{ CLASS_PRIEST,
	130, "Word of Recall",		30,
	spell_spirit_guide,	TAR_CHAR_SELF,	POS_STANDING,
	&gsn_spirit_guide,			SLOT(130),	35,	12, NULL, NULL
    },
	
	{ CLASS_MAGE,
	401, "Black Ward I",		1,
	spell_elemental_defense_i,	TAR_CHAR_DEFENSIVE,	POS_STANDING,
	&gsn_elemental_defense_i,			SLOT(401),	35,	12,
	"A slight silvery glow fades from you.",
	"A slight silvery glow fades from $n."
    },

	{ CLASS_MAGE,
	901, "Shock Bolt",		1,
	spell_shock_bolt,	TAR_CHAR_OFFENSIVE,	POS_STANDING,
	NULL,			SLOT(901),	35,	12, NULL, NULL
    },
	
	{ CLASS_SORCERER,
	701, "Blood Rune",		1,
	spell_bleed,	TAR_CHAR_OFFENSIVE,	POS_STANDING,
	&gsn_bleed,			SLOT(701),	35,	12, NULL, NULL
    },
	
	{ CLASS_MAGE,
	902, "Sleep",		2,
	spell_sleep,	TAR_CHAR_OFFENSIVE,	POS_STANDING,
	&gsn_sleep,			SLOT(902),	35,	12, NULL, NULL
    },
	
	{ CLASS_SORCERER,
	702, "Disrupting Touch",		2,
	spell_disrupt,	TAR_CHAR_OFFENSIVE,	POS_STANDING,
	&gsn_disrupt,			SLOT(702),	35,	12, NULL, NULL
    },
	
	{ CLASS_MAGE,
	903, "Water Bolt",		3,
	spell_water_bolt,	TAR_CHAR_OFFENSIVE,	POS_STANDING,
	NULL,			SLOT(903),	35,	12, NULL, NULL
    },
	
	{ CLASS_MAGE,
	503, "Protective Ward",		3,
	spell_protective_ward,	TAR_CHAR_DEFENSIVE,	POS_STANDING,
	&gsn_protective_ward,			SLOT(503),	35,	12,
	"The bright sparks that were encircling you fly off in all directions, spinning off into nothingness.",
	"The bright sparks that were encircling $n fly off in all directions, spinning off into nothingness."
    },
	
	{ CLASS_MAGE,
	406, "Black Ward II",		6,
	spell_elemental_defense_ii,	TAR_CHAR_DEFENSIVE,	POS_STANDING,
	&gsn_elemental_defense_ii,			SLOT(406),	35,	12,
	"A bright silvery glow fades from you.",
	"A bright silvery glow fades from $n."
    },
	
	
	{ CLASS_MAGE,
	906, "Flamebolt",		6,
	spell_fire_bolt,	TAR_CHAR_OFFENSIVE,	POS_STANDING,
	NULL,			SLOT(906),	35,	12, NULL, NULL
    },
	
	{ CLASS_SORCERER,
	706, "Stun Rune",			6,
	spell_jolt,		TAR_CHAR_OFFENSIVE,	POS_STANDING,
	NULL,			SLOT(706),	35,	12,
	"",
	""
    },
	
	{ CLASS_SORCERER,
	707, "Death Ray",		7,
	spell_death_ray,	TAR_CHAR_OFFENSIVE,	POS_STANDING,
	&gsn_death_ray,			SLOT(707),	35,	12, NULL, NULL
    },
	
	{ CLASS_MAGE,
	509, "Strength of Giants",		9,
	spell_strength,	TAR_CHAR_DEFENSIVE,	POS_STANDING,
	&gsn_strength,			SLOT(509),	35,	12,
	"You feel weaker.",
	"$n looks weaker."
    },
	
	{ CLASS_MAGE,
	910, "Lightning Bolt",		10,
	spell_lightning_bolt,	TAR_CHAR_OFFENSIVE,	POS_STANDING,
	NULL,			SLOT(910),	35,	12, NULL, NULL
    },
	
	{ CLASS_SORCERER,
	710, "Mana Storm",		10,
	spell_maelstrom,	TAR_CHAR_SELF,	POS_STANDING,
	&gsn_maelstrom,			SLOT(710),	35,	12, NULL, NULL
    },
	
	{ CLASS_MAGE,
	111, "Fireball",		11,
	spell_fireball,		TAR_CHAR_OFFENSIVE,	POS_STANDING,
	NULL,			SLOT(111),	15,	12,
	"",
	""
    },
	
	{ CLASS_MAGE,
	414, "Black Ward III",		14,
	spell_elemental_defense_iii,	TAR_CHAR_DEFENSIVE,	POS_STANDING,
	&gsn_elemental_defense_iii,			SLOT(414),	35,	12,
	"A brilliant silvery glow fades from you.",
	"A brilliant silvery glow fades from $n."
    },
	
	{ CLASS_SORCERER,
	715, "Curse",		15,
	spell_curse,	TAR_EITHER,	POS_STANDING,
	&gsn_curse,			SLOT(715),	35,	12, 	
	"You feel less doomed.",
	"A dark shadow separates from $n and dissolves into nothingness."
    },
	
	{ CLASS_MAGE,
	917, "Invisibility",		17,
	spell_invis,	TAR_CHAR_DEFENSIVE,	POS_STANDING,
	&gsn_invis,			SLOT(917),	35,	12,
	"You are now visible.",
	"$n suddenly appears out of nowhere."
    },
	
	{ CLASS_MAGE,
	918, "Cone of Lightning",			18,
	spell_cone_of_lightning,		TAR_CHAR_OFFENSIVE,	POS_STANDING,
	NULL,			SLOT(918),	35,	12,
	"",
	""
    },
	
	{ CLASS_SORCERER,
	720, "Animate Corpse",		20,
	spell_animate_corpse,	TAR_CHAR_OFFENSIVE,	POS_STANDING,
	&gsn_animate_corpse,			SLOT(720),	35,	12, NULL, NULL
    },
	
	{ CLASS_MAGE,
	420, "Identify",		20,
	spell_identify,		TAR_OBJ_INV,	POS_STANDING,
	NULL,			SLOT(420),	15,	12,
	"",
	""
    },
	
	{ CLASS_MAGE,
	920, "Summon Familiar",		25,
	spell_summon_familiar,	TAR_CHAR_SELF,	POS_STANDING,
	NULL,			SLOT(920),	35,	12, NULL, NULL
    },
	
	{ CLASS_MAGE,
	925, "Enchant",		25,
	spell_enchant,	TAR_CHAR_SELF,	POS_STANDING,
	NULL,			SLOT(925),	35,	12, NULL, NULL
    },
	
	{ CLASS_SORCERER,
	725, "Summon Demon",		25,
	spell_summon_demon,	TAR_CHAR_SELF,	POS_STANDING,
	&gsn_summon_demon,			SLOT(725),	35,	12, NULL, NULL
    },
	
	{ CLASS_SORCERER,
	730, "Sorcerous Travel",		30,
	spell_sorcerous_travel,	TAR_CHAR_SELF,	POS_STANDING,
	&gsn_sorcerous_travel,			SLOT(730),	35,	12, NULL, NULL
    }
	
	

};


#if defined(macintosh)
#include <types.h>
#else
#include <sys/types.h>
#endif
#include <ctype.h>
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <time.h>
#include <unistd.h>
#include "merc.h"

#if !defined(macintosh)
extern	int	_filbuf		args( (FILE *) );
#endif



/*
 * Globals.
 */
HELP_DATA *		help_first;
HELP_DATA *		help_last;

SHOP_DATA *		shop_first;
SHOP_DATA *		shop_last;

CHAR_DATA *		char_free;
EXTRA_DESCR_DATA *	extra_descr_free;
NOTE_DATA *		note_free;
OBJ_DATA *		obj_free;
PC_DATA *		pcdata_free;
ROOM_INDEX_DATA *	room_free;
EXIT_DATA *		exit_free;
VERB_TRAP_DATA *	verb_trap_free;

char			bug_buf		[2*MAX_INPUT_LENGTH];
CHAR_DATA *		char_list;
char *			help_greeting;
char			log_buf		[2*MAX_INPUT_LENGTH];
KILL_DATA		kill_table	[MAX_LEVEL];
NOTE_DATA *		note_list;
OBJ_DATA *		object_list;
TIME_INFO_DATA		time_info;
WEATHER_DATA		weather_info;

PREDELAY_DATA *		predelay_free;

short			gsn_blindness;
short			gsn_charm_person;
short			gsn_curse;
short			gsn_invis;
short			gsn_mass_invis;
short			gsn_poison;

	short	gsn_armor_use;
	short	gsn_shield_use;
	
	short	gsn_edged_weapons;
	short	gsn_blunt_weapons;
	short	gsn_brawling;
	short	gsn_two_handed_weapons;
	short	gsn_ranged_weapons;
	short	gsn_thrown_weapons;
	short	gsn_polearm_weapons;
	
	short	gsn_ambush;
	short	gsn_two_weapon_combat;
	short	gsn_combat_maneuvers;
	short	gsn_multi_opponent_combat;
	short	gsn_physical_fitness;
	short	gsn_dodging;
	short	gsn_arcane_symbols;
	short	gsn_magic_item_use;
	short	gsn_spell_aiming;
	short	gsn_harness_power;
	short	gsn_elemental_mana_control;
	short	gsn_mental_mana_control;
	short	gsn_spirit_mana_control;
	short	gsn_spell_research;
	short	gsn_elemental_lore;
	short	gsn_spiritual_lore;
	short	gsn_sorcerous_lore;
	short	gsn_mental_lore;
	short	gsn_survival;
	short	gsn_disarming_traps;
	short	gsn_picking_locks;
	short	gsn_stalking_and_hiding;
	short	gsn_perception;
	short	gsn_climbing;
	short	gsn_swimming;
	short	gsn_first_aid;
	short	gsn_trading;
	short	gsn_pickpocketing;
	
	short	gsn_elemental_defense_i;
	short	gsn_elemental_defense_ii;
	short	gsn_elemental_defense_iii;
	short	gsn_protective_ward;
	short	gsn_strength;
	short	gsn_sleep;
	short	gsn_invis;
	
	// Sorcerer Guild
	short	gsn_animate_corpse;
	short	gsn_bleed;
	short	gsn_disrupt;
	short	gsn_maelstrom;
	short	gsn_curse;
	short	gsn_summon_demon;
	short	gsn_sorcerous_travel;
	
	short	gsn_spirit_warding_i;
	short	gsn_spirit_warding_ii;
	short	gsn_spirit_defense;
	short	gsn_spirit_barrier;
	short	gsn_spirit_shield;
	short	gsn_spell_shield;
	short	gsn_lesser_shroud;
		
	// Cleric Guild
	short	gsn_bless_weapon;
	short	gsn_resurrect;
	short	gsn_spirit_guide;
	short	gsn_transference;

	// Healer Guild
	short	gsn_heal_i;
	short	gsn_heal_ii;
	short	gsn_heal_iii;
	short	gsn_heal_scars_i;
	short	gsn_heal_scars_ii;
	short	gsn_heal_scars_iii;
	short	gsn_restore_health_i;
	short	gsn_restore_health_ii;
	
	short	gsn_hold_undead;
	short	gsn_destroy_undead;
	short	gsn_death_ray;


/*
 * Locals.
 */
MOB_INDEX_DATA *	mob_index_hash		[MAX_KEY_HASH];
OBJ_INDEX_DATA *	obj_index_hash		[MAX_KEY_HASH];
ROOM_INDEX_DATA *	room_index_hash		[MAX_KEY_HASH];
char *			string_hash		[MAX_KEY_HASH];

AREA_DATA *		area_first;
AREA_DATA *		area_last;

extern char *			string_space;
extern char *			top_string;
extern char			str_empty	[1];

int			top_affect;
int			top_area;
int			top_ed;
int			top_vt;
int			top_exit;
int			top_help;
int			top_mob_index;
int			top_obj_index;
int			top_reset;
int			top_room;
int			top_shop;



/*
 * Semi-locals.
 */
bool			fBootDb;
FILE *			fpArea;
char			strArea[MAX_INPUT_LENGTH];



/*
 * Local booting procedures.
 */
void	init_mm		args( ( void ) );

void	load_area	args( ( FILE *fp ) );
void	load_helps	args( ( FILE *fp ) );
void	load_mobiles	args( ( FILE *fp ) );
void	load_objects	args( ( FILE *fp ) );
void	load_resets	args( ( FILE *fp ) );
void	load_rooms	args( ( FILE *fp ) );
void	load_shops	args( ( FILE *fp ) );
void	load_specials	args( ( FILE *fp ) );
void	load_notes	args( ( void ) );

void	fix_exits	args( ( void ) );

void	reset_area	args( ( AREA_DATA * pArea ) );



/*
 * Big mama top level function.
 */
void boot_db( void )
{
    /*
     * Init some data space stuff.
     */
    {
	if ( ( string_space = calloc( 1, MAX_STRING ) ) == NULL )
	{
	    bug( "Boot_db: can't alloc %d string space.", MAX_STRING );
	    exit( 1 );
	}
	top_string	= string_space;
	fBootDb		= TRUE;
    }

    /*
     * Init random number generator.
     */
    {
	init_mm( );
    }

    /*
     * Set time and weather.
     */
    {
	int hour = get_current_hour();
	int minute = get_current_minute();
	
	     if ( hour <  5 ) weather_info.sunlight = SUN_DARK;
	else if ( hour <  6 ) weather_info.sunlight = SUN_RISE;
	else if ( hour < 19 ) weather_info.sunlight = SUN_LIGHT;
	else if ( hour < 20 ) weather_info.sunlight = SUN_SET;
	else                  weather_info.sunlight = SUN_DARK;

		// set today's weather
		
	weather_info.precip_start_hour = hour + number_range(0,23);
	
	if( weather_info.precip_start_hour > 23 )
		weather_info.precip_start_hour = weather_info.precip_start_hour - 23;	
	
	weather_info.precip_start_minute = minute + number_range(0,59);
	
	if( weather_info.precip_start_minute > 59 )
		weather_info.precip_start_minute = weather_info.precip_start_minute - 59;
	
	weather_info.precip_end_hour = weather_info.precip_start_hour + number_range(0,1);
	
	if( weather_info.precip_end_hour > 23 )
		weather_info.precip_end_hour = weather_info.precip_end_hour - 23;	
	
	weather_info.precip_end_minute = weather_info.precip_start_minute + number_range(0,59);
	
	if( weather_info.precip_end_minute > 59 )
	{
		weather_info.precip_end_hour++;
		weather_info.precip_end_minute = weather_info.precip_end_minute - 59;
	}
	
	weather_info.sky = SKY_CLOUDLESS;

	weather_info.todays_high = get_high_temp( );
	weather_info.tonights_low = get_low_temp( );

	weather_info.temp = get_current_temp( );
    }

    /*
     * Assign gsn's for spells which have them.
     */
	 
	 // and for all skills.
    {
	int sn;

	for ( sn = 0; sn < MAX_SPELL; sn++ )
	{
	    if ( spell_table[sn].pgsn != NULL )
		*spell_table[sn].pgsn = sn;
	}
	
	for ( sn = 0; sn < MAX_SKILL; sn++ )
	{
	    if ( skill_table[sn].pgsn != NULL )
		*skill_table[sn].pgsn = sn;
	}
    }


    /*
     * Read in all the area files.
     */
    {
	FILE *fpList;

	if ( ( fpList = fopen( AREA_LIST, "r" ) ) == NULL )
	{
	    perror( AREA_LIST );
	    exit( 1 );
	}

	for ( ; ; )
	{
	    strcpy( strArea, fread_word( fpList ) );
	    if ( strArea[0] == '$' )
		break;

	    if ( strArea[0] == '-' )
	    {
		fpArea = stdin;
	    }
	    else
	    {
		if ( ( fpArea = fopen( strArea, "r" ) ) == NULL )
		{
		    perror( strArea );
		    exit( 1 );
		}
	    }

	    for ( ; ; )
	    {
		char *word;

		if ( fread_letter( fpArea ) != '#' )
		{
		    bug( "Boot_db: # not found.", 0 );
		    exit( 1 );
		}

		word = fread_word( fpArea );

		     if ( word[0] == '$'               )                 break;
		else if ( !str_cmp( word, "AREA"     ) ) load_area    (fpArea);
		else if ( !str_cmp( word, "HELPS"    ) ) load_helps   (fpArea);
		else if ( !str_cmp( word, "MOBILES"  ) ) load_mobiles (fpArea);
		else if ( !str_cmp( word, "OBJECTS"  ) ) load_objects (fpArea);
		else if ( !str_cmp( word, "RESETS"   ) ) load_resets  (fpArea);
		else if ( !str_cmp( word, "ROOMS"    ) ) load_rooms   (fpArea);
		else if ( !str_cmp( word, "SHOPS"    ) ) load_shops   (fpArea);
		else if ( !str_cmp( word, "SPECIALS" ) ) load_specials(fpArea);
		else
		{
		    bug( "Boot_db: bad section name.", 0 );
		    exit( 1 );
		}
	    }

	    if ( fpArea != stdin )
		fclose( fpArea );
	    fpArea = NULL;
	}
	fclose( fpList );
    }

    /*
     * Fix up exits.
     * Declare db booting over.
     * Reset all areas once.
     * Load up the notes file.
     */
    {
	fix_exits( );
	fBootDb	= FALSE;
	area_update( );
	load_notes( );
    }
	
	//random shit here
	{
	init_maze();
	}

    return;
}



/*
 * Snarf an 'area' header line.
 */
void load_area( FILE *fp )
{
    AREA_DATA *pArea;
	int i = 0;

    pArea		= alloc_perm( sizeof(*pArea) );
    pArea->reset_first	= NULL;
    pArea->reset_last	= NULL;
    pArea->name		= fread_string( fp );
	pArea->builders	= fread_string( fp );
	pArea->filename	= fread_string( fp );
	
	pArea->vnum_start	= fread_number( fp );
    pArea->vnum_final	= fread_number( fp );
    pArea->reset_length	= 0;
    pArea->area_bits	= 0;
	
	for( i = 0; i < 10; i++ )
	pArea->desc[i]	= fread_string( fp );
	
	for( i = 0; i < 5; i++ )
	pArea->creatures[i]	= fread_number( fp );
	
    pArea->age		= 15;
    pArea->nplayer	= 0;
	pArea->ncritter	= 0;

    if ( area_first == NULL )
	area_first = pArea;
    if ( area_last  != NULL )
	area_last->next = pArea;
    area_last	= pArea;
    pArea->next	= NULL;

    top_area++;
    return;
}



/*
 * Snarf a help section.
 */
void load_helps( FILE *fp )
{
    HELP_DATA *pHelp;

    for ( ; ; )
    {
	pHelp		= alloc_perm( sizeof(*pHelp) );
	pHelp->level	= fread_number( fp );
	pHelp->keyword	= fread_string( fp );
	if ( pHelp->keyword[0] == '$' )
	    break;
	pHelp->text	= fread_string( fp );

	if ( !str_cmp( pHelp->keyword, "greeting" ) )
	    help_greeting = pHelp->text;

	if ( help_first == NULL )
	    help_first = pHelp;
	if ( help_last  != NULL )
	    help_last->next = pHelp;

	help_last	= pHelp;
	pHelp->next	= NULL;
	top_help++;
    }

    return;
}



/*
 * Snarf a mob section.
 */
void load_mobiles( FILE *fp )
{
    MOB_INDEX_DATA *pMobIndex;
	EQUIP_DATA *pEquip;

    for ( ; ; )
    {
	short vnum;
	char letter;
	int iHash;

	letter				= fread_letter( fp );
	if ( letter != '#' )
	{
	    bug( "Load_mobiles: # not found.", 0 );
	    exit( 1 );
	}

	vnum				= fread_number( fp );
	if ( vnum == 0 )
	    break;

	fBootDb = FALSE;
	if ( get_mob_index( vnum ) != NULL )
	{
	    bug( "Load_mobiles: vnum %d duplicated.", vnum );
	    exit( 1 );
	}
	fBootDb = TRUE;

	pMobIndex			= alloc_perm( sizeof(*pMobIndex) );
	pMobIndex->vnum			= vnum;
	pMobIndex->player_name		= fread_string( fp );
	pMobIndex->short_descr		= fread_string( fp );
	pMobIndex->long_descr		= fread_string( fp );
	pMobIndex->description		= fread_string( fp );

	pMobIndex->long_descr[0]	= UPPER(pMobIndex->long_descr[0]);
	pMobIndex->description[0]	= UPPER(pMobIndex->description[0]);

	pMobIndex->act			= fread_number( fp ) | ACT_IS_NPC;
	pMobIndex->affected_by		= fread_number( fp );
	pMobIndex->pShop		= NULL;
	//align
	fread_number( fp );
	letter				= fread_letter( fp );
	pMobIndex->level		= number_fuzzy( fread_number( fp ) );

	pMobIndex->as		= fread_number( fp );
	pMobIndex->ds			= fread_number( fp );
	pMobIndex->cs			= fread_number( fp );
	pMobIndex->td			= fread_number( fp );
				
	pMobIndex->sex			= fread_number( fp );

	pMobIndex->equipped		= NULL;
	
	if ( letter != 'S' )
	{
	    bug( "Load_mobiles: vnum %d non-S.", vnum );
	    exit( 1 );
	}
	
	for ( ; ; )
	{
	    letter = fread_letter( fp );

	    if ( letter == 'E' )
	    {
		pEquip = (EQUIP_DATA *) alloc_perm( sizeof( *pEquip ) );
		pEquip->item = fread_number( fp );
		pEquip->location = fread_number ( fp );
		pEquip->next = pMobIndex->equipped;
		pMobIndex->equipped = pEquip;
	    }
	    else if ( letter == 'G' )
	    {
		    pMobIndex->gold = fread_number( fp );
	    }
	    else
	    {
		ungetc( letter, fp );
		break;
	    }
	}

	iHash			= vnum % MAX_KEY_HASH;
	pMobIndex->next		= mob_index_hash[iHash];
	mob_index_hash[iHash]	= pMobIndex;
	top_mob_index++;
	kill_table[pMobIndex->level].number++;
    }

    return;
}



/*
 * Snarf an obj section.
 */
void load_objects( FILE *fp )
{
    OBJ_INDEX_DATA *pObjIndex;

    for ( ; ; )
    {
	short vnum;
	char letter;
	int iHash;

	letter				= fread_letter( fp );
	if ( letter != '#' )
	{
	    bug( "Load_objects: # not found.", 0 );
	    exit( 1 );
	}

	vnum				= fread_number( fp );
	if ( vnum == 0 )
	    break;

	fBootDb = FALSE;
	if ( get_obj_index( vnum ) != NULL )
	{
	    bug( "Load_objects: vnum %d duplicated.", vnum );
	    exit( 1 );
	}
	fBootDb = TRUE;

	pObjIndex			= alloc_perm( sizeof(*pObjIndex) );
	pObjIndex->vnum			= vnum;
	pObjIndex->name			= fread_string( fp );
	pObjIndex->short_descr		= fread_string( fp );
	pObjIndex->description		= fread_string( fp );
	/* Action description */	  fread_string( fp );

/*
	pObjIndex->short_descr[0]	= LOWER(pObjIndex->short_descr[0]);
	pObjIndex->description[0]	= UPPER(pObjIndex->description[0]); */

	pObjIndex->item_type		= fread_number( fp );
	pObjIndex->extra_flags		= fread_number( fp );
	pObjIndex->wear_flags		= fread_number( fp );
	pObjIndex->value[0]		= fread_number( fp );
	pObjIndex->value[1]		= fread_number( fp );
	pObjIndex->value[2]		= fread_number( fp );
	pObjIndex->value[3]		= fread_number( fp );
	pObjIndex->weight		= fread_number( fp );
	pObjIndex->cost			= fread_number( fp );	/* Unused */
	pObjIndex->st 			=  fread_number( fp ); 
	pObjIndex->du 			=  fread_number( fp );
	
	if ( pObjIndex->item_type == ITEM_POTION )
	    SET_BIT(pObjIndex->extra_flags, ITEM_NODROP);

	for ( ; ; )
	{
	    char letter;

	    letter = fread_letter( fp );

	    if ( letter == 'A' )
	    {
		AFFECT_DATA *paf;

		paf			= alloc_perm( sizeof(*paf) );
		paf->type		= -1;
		paf->duration		= -1;
		paf->location		= fread_number( fp );
		paf->modifier		= fread_number( fp );
		paf->bitvector		= 0;
		paf->next		= pObjIndex->affected;
		pObjIndex->affected	= paf;
		top_affect++;
	    }

	    else if ( letter == 'E' )
	    {
		EXTRA_DESCR_DATA *ed;

		ed			= alloc_perm( sizeof(*ed) );
		ed->keyword		= fread_string( fp );
		ed->description		= fread_string( fp );
		ed->next		= pObjIndex->extra_descr;
		pObjIndex->extra_descr	= ed;
		top_ed++;
	    }
		
		else if ( letter == 'V' )
	    {
		//fixme add custom verb traps
		
		VERB_TRAP_DATA *vt;

		vt			= alloc_perm( sizeof(*vt) );
		vt->verb		= fread_string( fp );
		vt->firstPersonMessage		= fread_string( fp );
		vt->roomMessage		= fread_string( fp );
		vt->GSL		= fread_string( fp );
		vt->next		= pObjIndex->verb_trap;
		pObjIndex->verb_trap	= vt;
		top_vt++;

	    }

	    else
	    {
		ungetc( letter, fp );
		break;
	    }
	}

	/*
	 * Translate spell "slot numbers" to internal "skill numbers."
	 */
	switch ( pObjIndex->item_type )
	{
	case ITEM_PILL:
	case ITEM_POTION:
	case ITEM_SCROLL:
	    pObjIndex->value[1] = slot_lookup( pObjIndex->value[1] );
	    pObjIndex->value[2] = slot_lookup( pObjIndex->value[2] );
	    pObjIndex->value[3] = slot_lookup( pObjIndex->value[3] );
	    break;

	case ITEM_STAFF:
	case ITEM_WAND:
	    pObjIndex->value[3] = slot_lookup( pObjIndex->value[3] );
	    break;
	}

	iHash			= vnum % MAX_KEY_HASH;
	pObjIndex->next		= obj_index_hash[iHash];
	obj_index_hash[iHash]	= pObjIndex;
	top_obj_index++;
    }

    return;
}



/*
 * Snarf a reset section.
 */
void load_resets( FILE *fp )
{
    RESET_DATA *pReset;
	
	//printf("load_resets called\r\n");

    if ( area_last == NULL )
    {
	bug( "Load_resets: no #AREA seen yet.", 0 );
	exit( 1 );
    }

    for ( ; ; )
    {
	ROOM_INDEX_DATA *pRoomIndex;
	EXIT_DATA *pexit;
	char letter;

	if ( ( letter = fread_letter( fp ) ) == 'S' )
	    break;

	if ( letter == '*' )
	{
	    fread_to_eol( fp );
	    continue;
	}

	pReset		= alloc_perm( sizeof(*pReset) );
	pReset->command	= letter;
	/* if_flag */	  fread_number( fp );
	pReset->arg1	= fread_number( fp );
	pReset->arg2	= fread_number( fp );
	pReset->arg3	= (letter == 'G' || letter == 'R')
			    ? 0 : fread_number( fp );
			  fread_to_eol( fp );

	/*
	 * Validate parameters.
	 * We're calling the index functions for the side effect.
	 */
	switch ( letter )
	{
	default:
	    bug( "Load_resets: bad command '%c'.", letter );
	    exit( 1 );
	    break;

	case 'M':
	    get_mob_index  ( pReset->arg1 );
	    get_room_index ( pReset->arg3 );
	    break;

	case 'O':
	    get_obj_index  ( pReset->arg1 );
	    get_room_index ( pReset->arg3 );
	    break;

	case 'P':
	    get_obj_index  ( pReset->arg1 );
	    get_obj_index  ( pReset->arg3 );
	    break;

	case 'G':
	case 'E':
	    get_obj_index  ( pReset->arg1 );
	    break;

	case 'D':
	    pRoomIndex = get_room_index( pReset->arg1 );

	    if ( pReset->arg2 < 0
	    ||   pReset->arg2 > 10
	    || ( pexit = pRoomIndex->exit[pReset->arg2] ) == NULL
	    || !IS_SET( pexit->exit_info, EX_ISDOOR ) )
	    {
		bug( "Load_resets: 'D': exit %d not door.", pReset->arg2 );
		exit( 1 );
	    }

	    if ( pReset->arg3 < 0 || pReset->arg3 > 2 )
	    {
		bug( "Load_resets: 'D': bad 'locks': %d.", pReset->arg3 );
		exit( 1 );
	    }

	    break;

	case 'R':
	    pRoomIndex		= get_room_index( pReset->arg1 );

	    if ( pReset->arg2 < 0 || pReset->arg2 > 6 )
	    {
		bug( "Load_resets: 'R': bad exit %d.", pReset->arg2 );
		exit( 1 );
	    }

	    break;
	}

	if ( area_last->reset_first == NULL )
	    area_last->reset_first	= pReset;
	if ( area_last->reset_last  != NULL )
	    area_last->reset_last->next	= pReset;

	area_last->reset_last	= pReset;
	pReset->next		= NULL;
	top_reset++;
    }

    return;
}



/*
 * Snarf a room section.
 */
void load_rooms( FILE *fp )
{
    ROOM_INDEX_DATA *pRoomIndex;

    if ( area_last == NULL )
    {
	bug( "Load_resets: no #AREA seen yet.", 0 );
	exit( 1 );
    }

    for ( ; ; )
    {
	short vnum;
	char letter;
	int door;
	int iHash;

	letter				= fread_letter( fp );
	if ( letter != '#' )
	{
	    bug( "Load_rooms: # not found.", 0 );
	    exit( 1 );
	}

	vnum				= fread_number( fp );
	if ( vnum == 0 )
	    break;

	fBootDb = FALSE;
	if ( get_room_index( vnum ) != NULL )
	{
	    bug( "Load_rooms: vnum %d duplicated.", vnum );
	    exit( 1 );
	}
	fBootDb = TRUE;

	pRoomIndex			= alloc_perm( sizeof(*pRoomIndex) );
	pRoomIndex->people		= NULL;
	pRoomIndex->contents		= NULL;
	pRoomIndex->extra_descr		= NULL;
	pRoomIndex->area		= area_last;
	pRoomIndex->vnum		= vnum;
	pRoomIndex->name		= fread_string( fp );
	pRoomIndex->description		= fread_string( fp );
	
	/* Area number */		  fread_number( fp );
	pRoomIndex->room_flags		= fread_number( fp );
	pRoomIndex->sector_type		= fread_number( fp );
	pRoomIndex->light		= 0;
	for ( door = 0; door <= 10; door++ )
	    pRoomIndex->exit[door] = NULL;

	for ( ; ; )
	{
	    letter = fread_letter( fp );
		//printf("%c",letter); //debug only fixme
	    if ( letter == 'S' )
		break;

	    if ( letter == 'D' )
	    {
		EXIT_DATA *pexit;
		int locks;

		door = fread_number( fp );
		if ( door < 0 || door > 10 )
		{
		    bug( "Fread_rooms: vnum %d has bad door number.", vnum );
		    exit( 1 );
		}

		pexit			= alloc_perm( sizeof(*pexit) );
		pexit->description	= fread_string( fp );
		pexit->keyword		= fread_string( fp );
		pexit->exit_info	= 0;
		locks			= fread_number( fp );
		pexit->key		= fread_number( fp );
		pexit->vnum		= fread_number( fp );

		switch ( locks )
		{
		case 1: pexit->exit_info = EX_ISDOOR;                break;
		case 2: pexit->exit_info = EX_ISDOOR | EX_PICKPROOF; break;
		}

		pRoomIndex->exit[door]	= pexit;
		top_exit++;
	    }
	    else if ( letter == 'E' )
	    {
		EXTRA_DESCR_DATA *ed;

		ed			= alloc_perm( sizeof(*ed) );
		ed->keyword		= fread_string( fp );
		ed->description		= fread_string( fp );
		ed->next		= pRoomIndex->extra_descr;
		pRoomIndex->extra_descr	= ed;
		top_ed++;
	    }
	    else
	    {
		bug( "Load_rooms: vnum %d has flag not 'DES'.", vnum );
		exit( 1 );
	    }
	}

	iHash			= vnum % MAX_KEY_HASH;
	pRoomIndex->next	= room_index_hash[iHash];
	room_index_hash[iHash]	= pRoomIndex;
	top_room++;
	pRoomIndex->zone_next = NULL;
	if (area_last->room_first == NULL)
	{
	    area_last->room_first = pRoomIndex;
	    area_last->room_last = pRoomIndex;
	}
	else
	{
	    area_last->room_last->zone_next = pRoomIndex;
	    area_last->room_last = pRoomIndex;
	}
    }

    return;
}



/*
 * Snarf a shop section.
 */
void load_shops( FILE *fp )
{
    SHOP_DATA *pShop;

    for ( ; ; )
    {
	MOB_INDEX_DATA *pMobIndex;
	int iTrade;

	pShop			= alloc_perm( sizeof(*pShop) );
	pShop->keeper		= fread_number( fp );
	if ( pShop->keeper == 0 )
	    break;
	for ( iTrade = 0; iTrade < MAX_TRADE; iTrade++ )
	    pShop->buy_type[iTrade]	= fread_number( fp );
	pShop->profit_buy	= fread_number( fp );
	pShop->profit_sell	= fread_number( fp );
	pShop->open_hour	= fread_number( fp );
	pShop->close_hour	= fread_number( fp );
				  fread_to_eol( fp );
	pMobIndex		= get_mob_index( pShop->keeper );
	pMobIndex->pShop	= pShop;

	if ( shop_first == NULL )
	    shop_first = pShop;
	if ( shop_last  != NULL )
	    shop_last->next = pShop;

	shop_last	= pShop;
	pShop->next	= NULL;
	top_shop++;
    }

    return;
}



/*
 * Snarf spec proc declarations.
 */
void load_specials( FILE *fp )
{
    for ( ; ; )
    {
	MOB_INDEX_DATA *pMobIndex;
	OBJ_INDEX_DATA *pObjIndex;
	ROOM_INDEX_DATA *pRoomIndex;
	char letter;

	switch ( letter = fread_letter( fp ) )
	{
	default:
	    bug( "Load_specials: letter '%c' not *MS.", letter );
	    exit( 1 );

	case 'S':
	    return;

	case '*':
	    break;

	case 'M':
		pMobIndex		= get_mob_index	( fread_number ( fp ) );
	    pMobIndex->spec_fun	= spec_lookup	( fread_word   ( fp ) );
	    if ( pMobIndex->spec_fun == 0 )
	    {
		bug( "Load_specials: 'M': vnum %d.", pMobIndex->vnum );
		exit( 1 );
	    }
	    break;
	    break;
		
	case 'O':
	    pObjIndex		= get_obj_index	( fread_number ( fp ) );
	    pObjIndex->obj_spec_name = strdup( fread_word( fp ) );
	    pObjIndex->spec_fun	= obj_spec_lookup ( pObjIndex->obj_spec_name );
		
	    if ( pObjIndex->spec_fun == 0 )
	    {
		bug( "Load_specials: 'O': vnum %d.", pObjIndex->vnum );
		exit( 1 );
	    }
		//else
		//printf("spec_fun for room %s\r\n", pObjIndex->short_descr );
	
	    break;
		
	case 'R':
	    pRoomIndex		= get_room_index	( fread_number ( fp ) );

		pRoomIndex->spec_fun = room_fun_lookup( fread_word   ( fp )  );	
	    if ( pRoomIndex->spec_fun == 0 )
	    {
		bug( "Load_specials: 'R': vnum %d.", pRoomIndex->vnum );
		exit( 1 );
	    }
		//else
		//printf("spec_fun for room %s\r\n", pRoomIndex->name );
	
	    break;
	}

	fread_to_eol( fp );
    }
}



/*
 * Snarf notes file.
 */
void load_notes( void )
{
    FILE *fp;
    NOTE_DATA *pnotelast;

    if ( ( fp = fopen( NOTE_FILE, "r" ) ) == NULL )
	return;

    pnotelast = NULL;
    for ( ; ; )
    {
	NOTE_DATA *pnote;
	char letter;

	do
	{
	    letter = getc( fp );
	    if ( feof(fp) )
	    {
		fclose( fp );
		return;
	    }
	}
	while ( isspace(letter) );
	ungetc( letter, fp );

	pnote		= alloc_perm( sizeof(*pnote) );

	if ( str_cmp( fread_word( fp ), "sender" ) )
	    break;
	pnote->sender	= fread_string( fp );

	if ( str_cmp( fread_word( fp ), "date" ) )
	    break;
	pnote->date	= fread_string( fp );

	if ( str_cmp( fread_word( fp ), "to" ) )
	    break;
	pnote->to_list	= fread_string( fp );

	if ( str_cmp( fread_word( fp ), "subject" ) )
	    break;
	pnote->subject	= fread_string( fp );

	if ( str_cmp( fread_word( fp ), "text" ) )
	    break;
	pnote->text	= fread_string( fp );

	if ( note_list == NULL )
	    note_list		= pnote;
	else
	    pnotelast->next	= pnote;

	pnotelast	= pnote;
    }

    strcpy( strArea, NOTE_FILE );
    fpArea = fp;
    bug( "Load_notes: bad key word.", 0 );
    exit( 1 );
    return;
}



/*
 * Translate all room exits from virtual to real.
 * Has to be done after all rooms are read in.
 * Check for bad reverse exits.
 */
void fix_exits( void )
{
    extern const short rev_dir [];
    char buf[MAX_STRING_LENGTH];
    ROOM_INDEX_DATA *pRoomIndex;
    ROOM_INDEX_DATA *to_room;
    EXIT_DATA *pexit;
    EXIT_DATA *pexit_rev;
    int iHash;
    int door;

    for ( iHash = 0; iHash < MAX_KEY_HASH; iHash++ )
    {
	for ( pRoomIndex  = room_index_hash[iHash];
	      pRoomIndex != NULL;
	      pRoomIndex  = pRoomIndex->next )
	{
	    bool fexit;

	    fexit = FALSE;
	    for ( door = 0; door <= 10; door++ )
	    {
		if ( ( pexit = pRoomIndex->exit[door] ) != NULL )
		{
		    fexit = TRUE;
		    if ( pexit->vnum <= 0 )
			pexit->to_room = NULL;
		    else
			pexit->to_room = get_room_index( pexit->vnum );
		}
	    }

	    if ( !fexit )
		SET_BIT( pRoomIndex->room_flags, ROOM_NO_MOB );
	}
    }

    for ( iHash = 0; iHash < MAX_KEY_HASH; iHash++ )
    {
	for ( pRoomIndex  = room_index_hash[iHash];
	      pRoomIndex != NULL;
	      pRoomIndex  = pRoomIndex->next )
	{
	    for ( door = 0; door <= 10; door++ )
	    {
		if ( ( pexit     = pRoomIndex->exit[door]       ) != NULL
		&&   ( to_room   = pexit->to_room               ) != NULL
		&&   ( pexit_rev = to_room->exit[rev_dir[door]] ) != NULL
		&&   pexit_rev->to_room != pRoomIndex )
		{
		    sprintf( buf, "Fix_exits: %d:%d -> %d:%d -> %d.",
			pRoomIndex->vnum, door,
			to_room->vnum,    rev_dir[door],
			(pexit_rev->to_room == NULL)
			    ? 0 : pexit_rev->to_room->vnum );
		    bug( buf, 0 );
		}
	    }
	}
    }

    return;
}

void generate_creature( AREA_DATA *pArea )
{   
	int vnum;
	ROOM_INDEX_DATA *room;
	CHAR_DATA *victim;
	char buf[MSL];
	bool success = FALSE;
	int num = pArea->creatures[number_range(0, 4)];
	
	if( num < 0 )
	return;
	
	// equipped, custom mobs.
	if( num > 80 )
	{
	return;
	 
	 while( success == FALSE )
		{
		vnum = number_range( pArea->vnum_start, pArea->vnum_final);
		room = get_room_index( vnum );
		
		if( room != NULL && !IS_SET( room->room_flags, ROOM_NO_MOB ) && !IS_SET( room->room_flags, ROOM_SAFE ) )
		{
		    victim = create_mobile( get_mob_index( vnum ) );
	
			char_to_room( victim, room );
			success = TRUE;
			
			act("$n just arrived.", victim, NULL, NULL, TO_ROOM );
			break;
		}
		}
		
		return;
	}	
	
	//gs3 style mobs
		while( success == FALSE )
		{
		vnum = number_range( pArea->vnum_start, pArea->vnum_final);
		room = get_room_index( vnum );
		
		if( room != NULL && !IS_SET( room->room_flags, ROOM_NO_MOB ) && !IS_SET( room->room_flags, ROOM_SAFE ) )
		{
		    victim = create_mobile( get_mob_index( 2 ) );
		
			strcpy( buf, "" );
			sprintf(buf, "%s", creature_table[num].name );	
			victim->name = str_dup(buf);
	
			strcpy( buf, "" );
			sprintf(buf, "%s %s", select_a_an( creature_table[num].name ), creature_table[num].name );	
			victim->short_descr = str_dup( buf );

			victim->level = creature_table[num].level;
			victim->class = creature_table[num].class;
			victim->as = creature_table[num].ob;
			victim->ds = creature_table[num].db;
	
			victim->cs = creature_table[num].cs;
			victim->td = creature_table[num].td;
	
			victim->act = creature_table[num].act;
			victim->act += ACT_IS_NPC;
			victim->act += ACT_AGGRESSIVE;
			victim->affected_by = creature_table[num].aff;
	
			victim->max_hit = creature_table[num].hp;
			victim->hit = victim->max_hit;	
			
			victim->max_mana = victim->level * 3;
			victim->mana = victim->max_mana;
			
			if( !IS_SET(victim->act, ACT_SEXLESS) )
			victim->sex = number_range(1,2);
			else
			victim->sex = 0;
			
			if (IS_SET(victim->act, ACT_ANIMAL) )
			victim->gold = 0;
		
			
		if( (!IS_SET(victim->act, ACT_ANIMAL ) && !IS_SET(victim->act, ACT_AUTOMATON) ) 
			&& number_bits( 2 ) // not always
			&& get_eq_char( victim, EQUIP_RIGHTHAND ) == NULL ) 
		{
		OBJ_DATA *weapon = create_random_weapon( victim->level );
		
		obj_to_char( weapon, victim );
		obj_to_free_hand( weapon, victim );
		}
	
	
			char_to_room( victim, room );
			success = TRUE;
			
			act("$n just arrived.", victim, NULL, NULL, TO_ROOM );
			break;
		}
		}
}

int get_area_level( AREA_DATA *pArea )
{
return 100; //fixme
}
		
/*
 * Repopulate areas periodically.
 */
 
void area_update( void )
{
    AREA_DATA *pArea;

    for ( pArea = area_first; pArea != NULL; pArea = pArea->next )
    {
	/*
	 * Check for PC's.
	 */
	 
	 
	 // generate a creature if there's players, it's a suitable area, 
	 // and the area is mostly empty of NPCs
	 
	 // max generated creates = number of rooms in area / 10 * number of players in area;
	if ( pArea->nplayer > 0 && pArea->creatures[0] != -1 )
	{
		if( pArea->ncritter < pArea->nplayer * number_of_rooms( pArea ) / 10 ) 
		generate_creature( pArea );
	}

	/*
	 * Check age and reset.
	 */
	if ( pArea->nplayer == 0 || pArea->age >= 100 )
	{
	    reset_area( pArea );
	    pArea->age = 0;
	}
    }

    return;
}



/*
 * Reset one area.
 */
void reset_area( AREA_DATA *pArea )
{
    RESET_DATA *pReset;
    CHAR_DATA *mob;
    bool last;
    int level;

    mob 	= NULL;
    last	= TRUE;
    level	= 0;
    for ( pReset = pArea->reset_first; pReset != NULL; pReset = pReset->next )
    {
	ROOM_INDEX_DATA *pRoomIndex;
	MOB_INDEX_DATA *pMobIndex;
	OBJ_INDEX_DATA *pObjIndex;
	OBJ_INDEX_DATA *pObjToIndex;
	EXIT_DATA *pexit;
	OBJ_DATA *obj;
	OBJ_DATA *obj_to;

	switch ( pReset->command )
	{
	default:
	    bug( "Reset_area: bad command %c.", pReset->command );
	    break;

	case 'M':
	    if ( ( pMobIndex = get_mob_index( pReset->arg1 ) ) == NULL )
	    {
		bug( "Reset_area: 'M': bad vnum %d.", pReset->arg1 );
		continue;
	    }

	    if ( ( pRoomIndex = get_room_index( pReset->arg3 ) ) == NULL )
	    {
		bug( "Reset_area: 'R': bad vnum %d.", pReset->arg3 );
		continue;
	    }

	    level = URANGE( 0, pMobIndex->level - 2, LEVEL_HERO );
	    if ( pMobIndex->count >= pReset->arg2 )
	    {
		last = FALSE;
		break;
	    }

	    mob = create_mobile( pMobIndex );

	    /*
	     * Check for pet shop.
	     */
	    {
		ROOM_INDEX_DATA *pRoomIndexPrev;
		pRoomIndexPrev = get_room_index( pRoomIndex->vnum - 1 );
		if ( pRoomIndexPrev != NULL
		&&   IS_SET(pRoomIndexPrev->room_flags, ROOM_PET_SHOP) )
		    SET_BIT(mob->act, ACT_PET);
	    }

	    //if ( room_is_dark( pRoomIndex ) )
		//SET_BIT(mob->affected_by, AFF_INFRARED);

	    char_to_room( mob, pRoomIndex );
	    level = URANGE( 0, mob->level - 2, LEVEL_HERO );
	    last  = TRUE;
	    break;

	case 'O':
	    if ( ( pObjIndex = get_obj_index( pReset->arg1 ) ) == NULL )
	    {
		bug( "Reset_area: 'O': bad vnum %d.", pReset->arg1 );
		continue;
	    }

	    if ( ( pRoomIndex = get_room_index( pReset->arg3 ) ) == NULL )
	    {
		bug( "Reset_area: 'R': bad vnum %d.", pReset->arg3 );
		continue;
	    }

	    if ( pArea->nplayer > 0
	    ||   count_obj_list( pObjIndex, pRoomIndex->contents ) > 0 )
	    {
		last = FALSE;
		break;
	    }

			//printf("calling C_O %d", pObjIndex->vnum); //fixme debug only
	    obj       = create_object( pObjIndex, number_fuzzy( level ) );
	    obj->cost = 0;
	    obj_to_room( obj, pRoomIndex );
	    last = TRUE;
	    break;

	case 'P':
	    if ( ( pObjIndex = get_obj_index( pReset->arg1 ) ) == NULL )
	    {
		bug( "Reset_area: 'P': bad vnum %d.", pReset->arg1 );
		continue;
	    }

	    if ( ( pObjToIndex = get_obj_index( pReset->arg3 ) ) == NULL )
	    {
		bug( "Reset_area: 'P': bad vnum %d.", pReset->arg3 );
		continue;
	    }

	    if ( pArea->nplayer > 0
	    || ( obj_to = get_obj_type( pObjToIndex ) ) == NULL
	    ||   obj_to->in_room == NULL
	    ||   count_obj_list( pObjIndex, obj_to->contains ) > 0 )
	    {
		last = FALSE;
		break;
	    }
			//printf("calling C_O %d", pObjIndex->vnum); //fixme debug only
	    obj = create_object( pObjIndex, number_fuzzy( obj_to->level ) );
	    obj_to_obj( obj, obj_to );
	    last = TRUE;
	    break;

	case 'G':
	case 'E':
	    if ( ( pObjIndex = get_obj_index( pReset->arg1 ) ) == NULL )
	    {
		bug( "Reset_area: 'E' or 'G': bad vnum %d.", pReset->arg1 );
		continue;
	    }

	    if ( !last )
		break;

	    if ( mob == NULL )
	    {
		bug( "Reset_area: 'E' or 'G': null mob for vnum %d.",
		    pReset->arg1 );
		last = FALSE;
		break;
	    }

	    if ( mob->pIndexData->pShop != NULL )
	    {
		int olevel;

		switch ( pObjIndex->item_type )
		{
		default:		olevel = 0;                      break;
		case ITEM_PILL:		olevel = number_range(  0, 10 ); break;
		case ITEM_POTION:	olevel = number_range(  0, 10 ); break;
		case ITEM_SCROLL:	olevel = number_range(  5, 15 ); break;
		case ITEM_WAND:		olevel = number_range( 10, 20 ); break;
		case ITEM_STAFF:	olevel = number_range( 15, 25 ); break;
		case ITEM_ARMOR:	olevel = number_range(  5, 15 ); break;
		case ITEM_WEAPON:	olevel = number_range(  5, 15 ); break;
		}

					printf("calling C_O %d", pObjIndex->vnum); //fixme debug only
		obj = create_object( pObjIndex, olevel );
		SET_BIT( obj->extra_flags, ITEM_INVENTORY );
	    }
	    else
	    {
						//printf("calling C_O %d", pObjIndex->vnum); //fixme debug only
		obj = create_object( pObjIndex, number_fuzzy( level ) );
	    }
	    obj_to_char( obj, mob );
	    
		if ( pReset->command == 'E' )
		{
		//fixme double-load
		
		if( get_eq_char( mob, pReset->arg3 ) == NULL )
		equip_char( mob, obj, pReset->arg3 );
	    }
		
		last = TRUE;
	    break;

	case 'D':
	    if ( ( pRoomIndex = get_room_index( pReset->arg1 ) ) == NULL )
	    {
		bug( "Reset_area: 'D': bad vnum %d.", pReset->arg1 );
		continue;
	    }

	    if ( ( pexit = pRoomIndex->exit[pReset->arg2] ) == NULL )
		break;

	    switch ( pReset->arg3 )
	    {
	    case 0:
		REMOVE_BIT( pexit->exit_info, EX_CLOSED );
		REMOVE_BIT( pexit->exit_info, EX_LOCKED );
		break;

	    case 1:
		SET_BIT(    pexit->exit_info, EX_CLOSED );
		REMOVE_BIT( pexit->exit_info, EX_LOCKED );
		break;

	    case 2:
		SET_BIT(    pexit->exit_info, EX_CLOSED );
		SET_BIT(    pexit->exit_info, EX_LOCKED );
		break;
	    }

	    last = TRUE;
	    break;

	case 'R':
	    if ( ( pRoomIndex = get_room_index( pReset->arg1 ) ) == NULL )
	    {
		bug( "Reset_area: 'R': bad vnum %d.", pReset->arg1 );
		continue;
	    }

	    {
		int d0;
		int d1;

		for ( d0 = 0; d0 < pReset->arg2 - 1; d0++ )
		{
		    d1                   = number_range( d0, pReset->arg2-1 );
		    pexit                = pRoomIndex->exit[d0];
		    pRoomIndex->exit[d0] = pRoomIndex->exit[d1];
		    pRoomIndex->exit[d1] = pexit;
		}
	    }
	    break;
	}
    }

    return;
}



/*
 * Create an instance of a mobile.
 */
CHAR_DATA *create_mobile( MOB_INDEX_DATA *pMobIndex )
{
    CHAR_DATA *mob;

    if ( pMobIndex == NULL )
    {
	bug( "Create_mobile: NULL pMobIndex.", 0 );
	exit( 1 );
    }

    if ( char_free == NULL )
    {
	mob		= alloc_perm( sizeof(*mob) );
    }
    else
    {
	mob		= char_free;
	char_free	= char_free->next;
    }

    clear_char( mob );
    mob->pIndexData	= pMobIndex;

    mob->name		= pMobIndex->player_name;
    mob->short_descr	= pMobIndex->short_descr;
    mob->long_descr	= pMobIndex->long_descr;
    mob->description	= pMobIndex->description;
    mob->spec_fun	= pMobIndex->spec_fun;

    mob->level		= pMobIndex->level;
	mob->as		= pMobIndex->as;
	mob->ds		= pMobIndex->ds;
	mob->cs		= pMobIndex->cs;
	mob->td		= pMobIndex->td;
    mob->act		= pMobIndex->act;
    mob->affected_by	= pMobIndex->affected_by;
    mob->sex		= pMobIndex->sex;

    mob->armor		= interpolate( mob->level, 100, -100 );


	if( mob->level > 0 )
		mob->max_hit	= mob->level * 10;
	else
		mob->max_hit = ( mob->as + mob->ds ) / 13;
	
	if( mob->max_hit < 10 )
		mob->max_hit = 10;
	
    mob->hit		= mob->max_hit;
	
	if( pMobIndex->gold )
		mob->gold = pMobIndex->gold;
	
	if ( pMobIndex->equipped != NULL )
    {
	OBJ_DATA *obj;
	EQUIP_DATA *pEquip;

	for ( pEquip = pMobIndex->equipped; pEquip != NULL; pEquip = pEquip->next )
	{	
					
	    obj = create_object( get_obj_index( pEquip->item ), 0 );

	    if ( obj == NULL )
		continue;

	    obj_to_char( obj, mob );

	    if ( pEquip->location != -1 )
	    {
		equip_char( mob, obj, pEquip->location );
	    }
	}
    }

    /*
     * Insert in list.
     */
    mob->next		= char_list;
    char_list		= mob;
    pMobIndex->count++;
    return mob;
}



/*
 * Create an instance of an object.
 */
OBJ_DATA *create_object( OBJ_INDEX_DATA *pObjIndex, int level )
{
    static OBJ_DATA obj_zero;
    OBJ_DATA *obj;

    if ( pObjIndex == NULL )
    {
	bug( "Create_object: NULL pObjIndex.", 0 );
	exit( 1 );
    }

    if ( obj_free == NULL )
    {
	obj		= alloc_perm( sizeof(*obj) );
    }
    else
    {
	obj		= obj_free;
	obj_free	= obj_free->next;
    }

    *obj		= obj_zero;
    obj->pIndexData	= pObjIndex;
    obj->in_room	= NULL;
    obj->level		= level;
    obj->wear_loc	= -1;

    obj->name		= pObjIndex->name;
    obj->short_descr	= pObjIndex->short_descr;
    obj->description	= pObjIndex->description;
    obj->item_type	= pObjIndex->item_type;
    obj->extra_flags	= pObjIndex->extra_flags;
    obj->wear_flags	= pObjIndex->wear_flags;
    obj->value[0]	= pObjIndex->value[0];
    obj->value[1]	= pObjIndex->value[1];
    obj->value[2]	= pObjIndex->value[2];
    obj->value[3]	= pObjIndex->value[3];
    obj->weight		= pObjIndex->weight;
    obj->cost		= pObjIndex->cost;
	obj->verb_trap	= pObjIndex->verb_trap;
	
	/*
	if( obj->verb_trap != NULL )
	{
		printf("loading %s's verb trap %s\r\n", obj->short_descr, obj->verb_trap->verb );
	}
	*/
	
    obj->obj_spec_name	= pObjIndex->obj_spec_name;
    obj->spec_fun	= pObjIndex->spec_fun;
	
	/*
	if( pObjIndex->spec_fun != NULL )
	obj->spec_fun	= &pObjIndex->spec_fun;
	*/
	
	/*
	if( obj->spec_fun )
	{
		printf("%s has spec_fun CREATE_OBJ\r\n", obj->short_descr);
	}
	*/
	
	obj->st			= pObjIndex->st;
	
	
	//support for standard breakage values in objs defined in area files
	//makes conversion much easier
	
	if( obj->st == 0 )
	{
	if( obj->item_type == ITEM_WEAPON )
		obj->st = weapon_table[obj->value[0]].st;

	if( obj->item_type == ITEM_ARMOR ) 
		obj->st = armor_table[obj->value[3]].st;
	
	if( obj->item_type == ITEM_SHIELD )
		obj->st = shield_table[obj->value[0]].st;
	}
	
	obj->du			= pObjIndex->du;
	
	if( obj->du == 0 )
	{
	if( obj->item_type == ITEM_WEAPON  ) 
		obj->du = weapon_table[obj->value[0]].du;

	if( obj->item_type == ITEM_ARMOR  ) 
		obj->du = armor_table[obj->value[3]].du;
	
	if( obj->item_type == ITEM_SHIELD )
		obj->st = shield_table[obj->value[0]].du;
	}
	
    obj->next		= object_list;
    object_list		= obj;
    pObjIndex->count++;
	
	obj_strings( obj ); //nice

    return obj;
}



/*
 * Clear a new character.
 */
void clear_char( CHAR_DATA *ch )
{
    static CHAR_DATA ch_zero;
	short i;

    *ch				= ch_zero;
    ch->name			= &str_empty[0];
    ch->short_descr		= &str_empty[0];
    ch->long_descr		= &str_empty[0];
    ch->description		= &str_empty[0];
	ch->predelay_time		= 0;
	ch->predelay_info		= NULL;
    ch->logon			= current_time;
    ch->armor			= 100;
    ch->position		= POS_STANDING;
    ch->practice		= 0;
    ch->hit				= 20;
    ch->max_hit			= 20;
    ch->mana			= 0;
    ch->max_mana		= 0;
    ch->move			= 0;
    ch->max_move		= 0;
	ch->as				= 0;
	ch->ds				= 0;
	ch->cs				= 0;
	ch->td				= 0;
	ch->decay			= 0;
	
	for( i = 0; i < MAX_BODY; i++ )
	{
		ch->body[i] = 0;
		ch->scars[i] = 0;
		ch->bleed[i] = 0;
		ch->bandage[i] = 0;
	}
	
    return;
}



/*
 * Free a character.
 */
void free_char( CHAR_DATA *ch )
{
    OBJ_DATA *obj;
    OBJ_DATA *obj_next;
    AFFECT_DATA *paf;
    AFFECT_DATA *paf_next;

    for ( obj = ch->carrying; obj != NULL; obj = obj_next )
    {
	obj_next = obj->next_content;
	extract_obj( obj );
    }

    for ( paf = ch->affected; paf != NULL; paf = paf_next )
    {
	paf_next = paf->next;
	affect_remove( ch, paf );
    }

    free_string( ch->name		);
    free_string( ch->short_descr	);
    free_string( ch->long_descr		);
    free_string( ch->description	);
	
	free_predelay( ch->predelay_info );

    if ( ch->pcdata != NULL )
    {
	free_string( ch->pcdata->pwd		);
	free_string( ch->pcdata->bamfin		);
	free_string( ch->pcdata->bamfout	);
	free_string( ch->pcdata->title		);
	ch->pcdata->next = pcdata_free;
	pcdata_free      = ch->pcdata;
    }

    ch->next	     = char_free;
    char_free	     = ch;
    return;
}


/*
 * free_exit and new_exit use to_room as a temporary next pointer.
 */
void free_exit( EXIT_DATA *exit )
{
    if ( exit == NULL )
	return;

    free_string( exit->description );
    free_string( exit->keyword );

    exit->to_room = (ROOM_INDEX_DATA *) exit_free;
    exit_free = exit;

    return;
}

void free_extra_descr( EXTRA_DESCR_DATA *ed )
{
    if ( ed == NULL )
	return;

    free_string( ed->keyword );
    free_string( ed->description );

    ed->next = extra_descr_free;
    extra_descr_free = ed;

    return;
}

void free_room( ROOM_INDEX_DATA *room_from )
{
    ROOM_INDEX_DATA *room_to;
    ROOM_INDEX_DATA *pRoomIndex;
    ROOM_INDEX_DATA *pRoom_next;
    CHAR_DATA *vch;
    EXTRA_DESCR_DATA *ed;
    int door;
    int iHash;

    room_to = get_room_index( ROOM_VNUM_CHAT );

    if ( room_from == room_to )
    {
	bug("Attempted free of chat room.", 0 );
	return;
    }

    for ( door = 0; door <= 10; door++ )
    {
	if ( room_from->exit[door] != NULL
	&& room_from->exit[door]->to_room != NULL
	&& room_from->exit[door]->to_room->area == room_from->area )
	{
	    room_to = room_from->exit[door]->to_room;
	    break;
	}
    }

    while ( room_from->contents != NULL )
		extract_obj( room_from->contents );

    for ( door = 0; door <= 10; door++ )
		free_exit( room_from->exit[door] );

    while ( room_from->people != NULL )
    {
	if ( IS_NPC( room_from->people ) )
	{
	    extract_char( room_from->people, TRUE );
	}
	else
	{
	    char_from_room( (vch = room_from->people) );
	    char_to_room( vch, room_to );
	    do_look( vch, "auto" );
	}
    }

    for ( iHash = 0; iHash < MAX_KEY_HASH; iHash++ )
    {
	if ( room_index_hash[iHash] == NULL )
	    continue;

	for ( pRoomIndex = room_index_hash[iHash]; pRoomIndex != NULL; pRoomIndex = pRoomIndex->next )
	{
	    for ( door = 0; door <= 10; door++ )
	    {
		if ( pRoomIndex->exit[door] != NULL )
		{
		    if ( pRoomIndex->exit[door]->to_room == room_from )
		    {
			pRoomIndex->exit[door]->to_room = NULL;
			pRoomIndex->exit[door]->vnum = 0;
			pRoomIndex->exit[door]->key = 0;

			if ( pRoomIndex->exit[door]->description[0] == '\0'
			&& pRoomIndex->exit[door]->keyword[0] == '\0' )
			{
			    free_exit( pRoomIndex->exit[door] );
			    pRoomIndex->exit[door] = NULL;
			}
		    }
		}
	    }
	}
    }

    iHash = room_from->vnum % MAX_KEY_HASH;
    for ( pRoomIndex = room_index_hash[iHash]; pRoomIndex != NULL; pRoomIndex = pRoom_next )
    {
	pRoom_next = pRoomIndex->next;

	if ( pRoomIndex == room_from )
	{
	    room_index_hash[iHash] = pRoomIndex->next;
	    break;
	}

	if ( pRoomIndex->next == room_from )
	{
	    pRoomIndex->next = room_from->next;
	    break;
	}
    }

	if ( room_from->area->room_first == room_from )
	{
	room_from->area->room_first = room_from->zone_next;
	if ( room_from->zone_next == NULL )
	    room_from->area->room_last = NULL;
	}
	else
    for ( pRoomIndex = room_from->area->room_first; pRoomIndex != NULL; pRoomIndex = pRoom_next )
    {
	pRoom_next = pRoomIndex->zone_next;

	if ( pRoom_next == room_from )
	{
	    pRoomIndex->zone_next = pRoom_next->zone_next;
	    
		if ( room_from->zone_next == NULL )
			room_from->area->room_last = pRoomIndex;
	    
		break;
	}
    }

    free_string( room_from->name );
    free_string( room_from->description );

    while ( room_from->extra_descr != NULL )
    {
	ed = room_from->extra_descr;

	room_from->extra_descr = ed->next;

	free_extra_descr( ed );
    }

    room_from->next = room_free;
    room_free = room_from;

    return;
}


void free_predelay( PREDELAY_DATA *p )
{
    if ( p == NULL )
	return;

    p->fnptr = (DO_FUN *) predelay_free;
    predelay_free = p;

    return;
}


PREDELAY_DATA *new_predelay( void )
{
    PREDELAY_DATA *p;
	
	//printf("new predelay %d\r\n", 0 );

    if ( predelay_free == NULL )
    {
	p                  = alloc_perm( sizeof(*p) );
    }
    else
    {
	p                  = predelay_free;
	predelay_free      = (PREDELAY_DATA *) predelay_free->fnptr;
    }

    return p;
}


/*
 * Get an extra description from a list.
 */
char *get_extra_descr( const char *name, EXTRA_DESCR_DATA *ed )
{
    for ( ; ed != NULL; ed = ed->next )
    {
	if ( is_name( name, ed->keyword ) )
	    return ed->description;
    }
    return NULL;
}



/*
 * Translates mob virtual number to its mob index struct.
 * Hash table lookup.
 */
MOB_INDEX_DATA *get_mob_index( int vnum )
{
    MOB_INDEX_DATA *pMobIndex;

    for ( pMobIndex  = mob_index_hash[vnum % MAX_KEY_HASH];
	  pMobIndex != NULL;
	  pMobIndex  = pMobIndex->next )
    {
	if ( pMobIndex->vnum == vnum )
	    return pMobIndex;
    }

    if ( fBootDb )
    {
	bug( "Get_mob_index: bad vnum %d.", vnum );
	exit( 1 );
    }

    return NULL;
}



/*
 * Translates mob virtual number to its obj index struct.
 * Hash table lookup.
 */
OBJ_INDEX_DATA *get_obj_index( int vnum )
{
    OBJ_INDEX_DATA *pObjIndex;

    for ( pObjIndex  = obj_index_hash[vnum % MAX_KEY_HASH];
	  pObjIndex != NULL;
	  pObjIndex  = pObjIndex->next )
    {
	if ( pObjIndex->vnum == vnum )
	    return pObjIndex;
    }

    if ( fBootDb )
    {
	bug( "Get_obj_index: bad vnum %d.", vnum );
	exit( 1 );
    }

    return NULL;
}



/*
 * Translates mob virtual number to its room index struct.
 * Hash table lookup.
 */
ROOM_INDEX_DATA *get_room_index( int vnum )
{
    ROOM_INDEX_DATA *pRoomIndex;

    for ( pRoomIndex  = room_index_hash[vnum % MAX_KEY_HASH];
	  pRoomIndex != NULL;
	  pRoomIndex  = pRoomIndex->next )
    {
	if ( pRoomIndex->vnum == vnum )
	    return pRoomIndex;
    }

    if ( fBootDb )
    {
	bug( "Get_room_index: bad vnum %d.", vnum );
	exit( 1 );
    }

    return NULL;
}


void show_connections (CHAR_DATA *ch, AREA_DATA *pArea)
{
  ROOM_INDEX_DATA *pRoomIndex;
  int d;
  
	// each room
    for ( pRoomIndex = pArea->room_first; pRoomIndex != NULL; pRoomIndex = pRoomIndex->zone_next )
	{	
		// each exit
		for ( d = 0; d < 12; d++)
		if (pRoomIndex->exit[d]->to_room->area != pRoomIndex->area )
		{
			
			if( pRoomIndex->exit[d] != NULL && pRoomIndex->exit[d]->to_room != NULL )
			{
			printf_to_char( ch, "%s (VNUM %d) connects to %s (VNUM %d)\r\n", 
			pRoomIndex->area->name, 
			pRoomIndex->vnum,
			pRoomIndex->exit[d]->to_room->area->name,
			pRoomIndex->exit[d]->to_room->vnum);
			}
		}
	}

		
   return;
}

int number_of_rooms( AREA_DATA *pArea )
{
  ROOM_INDEX_DATA *pRoomIndex;
  int i = 0;
  
    for ( pRoomIndex = pArea->room_first; pRoomIndex != NULL; pRoomIndex = pRoomIndex->zone_next ) 
		i++;
		
   return i;
}

void do_areas( CHAR_DATA *ch, char *argument )
{
    AREA_DATA *pArea1;
	
	if( !str_cmp( argument, "list" ) && IS_IMMORTAL(ch) )
	{
	int rooms = 0;
	
	for ( pArea1 = area_first; pArea1 != NULL; pArea1 = pArea1->next )
    {
	if (str_cmp(pArea1->name, "Limbo"))
	{
	printf_to_char( ch, "%-12s %-20s %8d %8d %2d (%d rooms)\r\n",
	    pArea1->filename,
	    pArea1->name,
	    pArea1->vnum_start,
	    pArea1->vnum_final,
	    pArea1->area_bits,
		number_of_rooms(pArea1));
		
	//show_connections( ch, pArea1 );
	}
	
	if (str_cmp(pArea1->name, "Limbo"))
	rooms += number_of_rooms(pArea1);
    }
	
	printf_to_char( ch, "%d rooms total.\r\n", rooms );
	return;
	}
	
	// total creatures in area, total rooms
	// creatures that spawn, their levels
	
			if(IS_IMMORTAL(ch))
			{
				printf_to_char(ch, "Area: %s\r\nBuilders: %s\r\n", ch->in_room->area->name,
					ch->in_room->area->builders  );
			
				printf_to_char( ch, "Room: %d of %d.\r\n",
					ch->in_room->vnum - ch->in_room->area->vnum_start,
					number_of_rooms( ch->in_room->area ) );
			
			printf_to_char(ch, "Vnums: %d to %d\r\nNumber of creatures: %d\r\nNumber of players: %d\r\nAge of area: %d\r\nCreatures: %s (%d) | %s (%d) | %s (%d) | %s (%d) | %s (%d)\r\n", 
			ch->in_room->area->vnum_start,
			ch->in_room->area->vnum_final,
			ch->in_room->area->ncritter, ch->in_room->area->nplayer, ch->in_room->area->age,
			ch->in_room->area->creatures[0] == -1 ? "---" : creature_table[ch->in_room->area->creatures[0]].name, 
			ch->in_room->area->creatures[0] == -1 ? 0 :creature_table[ch->in_room->area->creatures[0]].level,
			ch->in_room->area->creatures[1] == -1 ? "---" : creature_table[ch->in_room->area->creatures[1]].name, 
			ch->in_room->area->creatures[1] == -1 ? 0 :creature_table[ch->in_room->area->creatures[1]].level,
			ch->in_room->area->creatures[2] == -1 ? "---" : creature_table[ch->in_room->area->creatures[2]].name, 
			ch->in_room->area->creatures[2] == -1 ? 0 :creature_table[ch->in_room->area->creatures[2]].level,
			ch->in_room->area->creatures[3] == -1 ? "---" : creature_table[ch->in_room->area->creatures[3]].name, 
			ch->in_room->area->creatures[3] == -1 ? 0 :creature_table[ch->in_room->area->creatures[3]].level,
			ch->in_room->area->creatures[4] == -1 ? "---" : creature_table[ch->in_room->area->creatures[4]].name, 
			ch->in_room->area->creatures[4] == -1 ? 0 :creature_table[ch->in_room->area->creatures[4]].level);	
			}
			else
			{
			printf_to_char( ch, "You carefully survey your surroundings and guess that your current location is %s.\r\n",ch->in_room->area->name );
			}
						

    return;
}

#define MAX_COLOR_LIST             18
#define MAX_CLOTH_LIST              7

char *   const  color_list [MAX_COLOR_LIST] =
{
    "red",
    "blue",
    "black",
    "turquoise",
    "yellow",
    "faded indigo",
    "purple",
    "magenta",
    "lavender",
    "green",
    "white",
    "bright orange",
    "violet",
    "maroon",
    "mauve",
    "brown",          /* 16 */
    "tan",
    "grey"
};

char *   const  cloth_list [MAX_CLOTH_LIST] =
{
    "satin",
    "linen",
    "silk",
    "cloth",
    "wool",
    "cotton",
    "reed"            /* 7 */
};


void obj_strings( OBJ_DATA *obj )
{
   char buf[MAX_STRING_LENGTH];
   const char *str;
   const char *i;
   char *point;
   int pass;

   char c_str[256];
   char l_str[256];

   sprintf( c_str, "%s", color_list[number_range( 0, MAX_COLOR_LIST-1 )] );
   sprintf( l_str, "%s", cloth_list[number_range( 0, MAX_CLOTH_LIST-1 )] );

   for ( pass = 0; pass < 3; pass++ )
   {
   switch( pass )
   {
      default: str = STR( obj, short_descr );
	       bug( "obj_strings: bad pass %d", pass );   break;
       case 0: str = STR( obj, name );               break;
       case 1: str = STR( obj, short_descr ); break;
       case 2: str = STR( obj, description ); break;
   }

   point = buf;

   while( *str != '\0' )
   {
      if( *str != '$' )
      {
	 *point++ = *str++;
	 continue;
      }
      ++str;
      switch( *str )
      {
	 default : i = " "; break;
	 case 'c': i = c_str; break;
	 case 'l': i = l_str; break;
      } 
      ++str;
      while( (*point = *i) != '\0' )
	 ++point, ++i;      
   }

   *point = '\0';

   switch( pass )
   {
      default: break;
       case 0: obj->name        = str_dup( buf ); break;
       case 1: obj->short_descr = str_dup( buf ); break;
       case 2: obj->description = str_dup( buf ); break;
   }
   }
   return;
}


/*
 * Stick a little fuzz on a number.
 */
int number_fuzzy( int number )
{
	// let's not and say we did. we're precise here at grindstone.
	
	return number;
	
	/*
    switch ( number_bits( 2 ) )
    {
    case 0:  number -= 1; break;
    case 3:  number += 1; break;
    }

    return UMAX( 1, number ); */
}



/*
 * Generate a random number.
 */
int number_range( int from, int to )
{
    int power;
    int number;

    if ( ( to = to - from + 1 ) <= 1 )
	return from;

    for ( power = 2; power < to; power <<= 1 )
	;

    while ( ( number = number_mm( ) & (power - 1) ) >= to )
	;

    return from + number;
}



/*
 * Generate a percentile roll.
 */
int number_percent( void )
{
    int percent;

    while ( ( percent = number_mm( ) & (128-1) ) > 99 )
	;

    return 1 + percent;
}



/*
 * Generate a random door.
 */
int number_door( void )
{
    return number_range(1,9);
}



int number_bits( int width )
{
    return number_mm( ) & ( ( 1 << width ) - 1 );
}


/*
 * I've gotten too many bad reports on OS-supplied random number generators.
 * This is the Mitchell-Moore algorithm from Knuth Volume II.
 * Best to leave the constants alone unless you've read Knuth.
 * -- Furey
 */

/* I noticed streaking with this random number generator, so I switched
   back to the system srandom call.  If this doesn't work for you, 
   define OLD_RAND to use the old system -- Alander */

#if defined (OLD_RAND)
static  int     rgiState[2+55];
#endif
 
void init_mm( )
{
#if defined (OLD_RAND)
    int *piState;
    int iState;
 
    piState     = &rgiState[2];
 
    piState[-2] = 55 - 55;
    piState[-1] = 55 - 24;
 
    piState[0]  = ((int) current_time) & ((1 << 30) - 1);
    piState[1]  = 1;
    for ( iState = 2; iState < 55; iState++ )
    {
        piState[iState] = (piState[iState-1] + piState[iState-2])
                        & ((1 << 30) - 1);
    }
#else
    srandom(time(NULL)^getpid());
#endif
    return;
}
 

long number_mm( void )
{
#if defined (OLD_RAND)
    int *piState;
    int iState1;
    int iState2;
    int iRand;
 
    piState             = &rgiState[2];
    iState1             = piState[-2];
    iState2             = piState[-1];
    iRand               = (piState[iState1] + piState[iState2])
                        & ((1 << 30) - 1);
    piState[iState1]    = iRand;
    if ( ++iState1 == 55 )
        iState1 = 0;
    if ( ++iState2 == 55 )
        iState2 = 0;
    piState[-2]         = iState1;
    piState[-1]         = iState2;
    return iRand >> 6;
#else
    return random() >> 6;
#endif
}



/*
 * Roll some dice.
 */
int dice( int number, int size )
{
    int idice;
    int sum;

    switch ( size )
    {
    case 0: return 0;
    case 1: return number;
    }

    for ( idice = 0, sum = 0; idice < number; idice++ )
	sum += number_range( 1, size );

    return sum;
}



/*
 * Simple linear interpolation.
 */
int interpolate( int level, int value_00, int value_32 )
{
    return value_00 + level * (value_32 - value_00) / 32;
}


/*
 * This function is here to aid in debugging.
 * If the last expression in a function is another function call,
 *   gcc likes to generate a JMP instead of a CALL.
 * This is called "tail chaining."
 * It hoses the debugger call stack for that call.
 * So I make this the last call in certain critical functions,
 *   where I really need the call stack to be right for debugging!
 *
 * If you don't understand this, then LEAVE IT ALONE.
 * Don't remove any calls to tail_chain anywhere.
 *
 * -- Furey
 */
void tail_chain( void )
{
    return;
}


#if defined(macintosh)
#include <types.h>
#else
#include <sys/types.h>
#endif
#include <stdio.h>
#include <string.h>
#include <time.h>
#include <unistd.h>
#include "merc.h"



/*
 * Local functions.
 */


void	dam_message	args( ( CHAR_DATA *ch, CHAR_DATA *victim, int dam,
			    int dt, int loc ) );
void	death_cry	args( ( CHAR_DATA *ch ) );
void	group_gain	args( ( CHAR_DATA *ch, CHAR_DATA *victim ) );
int	xp_compute	args( ( CHAR_DATA *gch, CHAR_DATA *victim ) );
void	make_corpse	args( ( CHAR_DATA *ch ) );
void	raw_kill	args( ( CHAR_DATA *victim ) );
void	disarm		args( ( CHAR_DATA *ch, CHAR_DATA *victim ) );
void	trip		args( ( CHAR_DATA *ch, CHAR_DATA *victim ) );



char *combat_string ( int number );

char *combat_string2 ( int number );

char *difference_string ( int number );



void multi_hit( CHAR_DATA *ch, CHAR_DATA *victim, int dt )
{
    one_hit( ch, victim, dt, 0 );
    return;
}

int get_melee_as( CHAR_DATA *ch )
{
	return ch->as; //fixme avds
}

int get_melee_ds( CHAR_DATA *ch )
{
	return ch->ds;
}


char *luck_string ( int number )
{
	if ( number < 10 )
		return "doomed";	
	else if ( number < 25 )
		return "unlucky";		
	else if ( number < 74 )
		return "average";
	else if ( number < 89 )
		return "lucky";
	else
		return "fortunate";
}		

char *combat_string ( int number )
{
	if ( number < 25 )
		return "pitiful";
		
	else if ( number < 39 )
		return "awful";

	else if ( number < 59 )
		return "weak";	

	else if ( number < 90 )
		return "mediocre";	

	else if ( number < 115 )
		return "fair";

	else if ( number < 144 )
		return "decent";	

	else if ( number < 200 )
		return "solid";		

	else if ( number < 250 )
		return "strong";

	else if ( number < 300 )
		return "great";

	else if ( number < 350 )
		return "excellent";	

	else if ( number < 400 )
		return "fearsome";	
		
	else if ( number < 450 )
		return "vicious";	
		
	else if ( number < 500 )
		return "awesome";	

	else
		return "immense";			
}



char *combat_string2 ( int number )
{
	if ( number < 25 )
		return "vulnerable";
		
	else if ( number < 50 )
		return "exposed";

	else if ( number < 65 )
		return "poor";	

	else if ( number < 95 )
		return "bad";	

	else if ( number < 125 )
		return "pretty bad";

	else if ( number < 145 )
		return "middling";	

	else if ( number < 195 )
		return "hardy";		

	else if ( number < 245 )
		return "resistent";

	else if ( number < 325 )
		return "tough";

	else if ( number < 375)
		return "marvelous";	

	else if ( number < 400 )
		return "outstanding";	
		
	else if ( number < 450 )
		return "splendid";	
		
	else if ( number < 500 )
		return "spectacular";	

	else
		return "invulnerable";			
}

char * difference_string( int number )
{
	if ( number < -100 )
		return "cannot hope to match";
		
	else if ( number < -50 )
		return "crumbles before";

	else if ( number < 0 )
		return "fails to overcome";	

	else if ( number < 50 )
		return "penetrates";	

	else
		return "overpowers";
}

// fixme i got lazy and don't want to add avds for every weapon vs every armor..

//so ADVANTAGE is born, lol

short get_adv( CHAR_DATA *ch, CHAR_DATA *victim )
{
	short adv = 20; // a closed fist
	OBJ_DATA *wield = get_eq_char( ch, EQUIP_RIGHTHAND );
	
	if( wield != NULL && wield->item_type == ITEM_WEAPON )
	{
	if( weapon_table[wield->value[0]].weapon_type == WEAPON_TYPE_EDGED )
		adv += 10;
	
	else if( weapon_table[wield->value[0]].weapon_type == WEAPON_TYPE_BLUNT )
		adv += 10;
	
	else if( weapon_table[wield->value[0]].weapon_type == WEAPON_TYPE_POLEARM )
		adv += 20;

	else if( weapon_table[wield->value[0]].weapon_type == WEAPON_TYPE_TWOHANDED  )
		adv += 20;
	}
	
	wield = get_eq_char( victim, EQUIP_CHEST );
	
	if( wield != NULL && wield->item_type == ITEM_ARMOR )
	{
		if( armor_table[wield->value[3]].v0 == ARMOR_GROUP_LEATHER )
			adv -= 5;
		else if( armor_table[wield->value[3]].v0 == ARMOR_GROUP_SCALE )
			adv -= 10;
		else if( armor_table[wield->value[3]].v0 == ARMOR_GROUP_CHAIN )
			adv -= 15;
		else if( armor_table[wield->value[3]].v0 == ARMOR_GROUP_PLATE )
			adv -= 20;
	}

	return adv;
}


/*
Invar falchion clashes with reinforced leather!
[STR/DU: 80/175 vs. 35/270, d100(Open)= 120]
*/

bool breakage_check( CHAR_DATA *ch, OBJ_DATA *breaker, CHAR_DATA *victim, OBJ_DATA *breakee )
{
	char buf[MAX_STRING_LENGTH];
	bool broke = FALSE;
	short roll = open_1d100();
	
	//printf("breakage check called: %s is breaker, %s is breakee.\r\n", breaker->short_descr, breakee->short_descr );
	
	if( ch == NULL || victim == NULL )
		return FALSE;
	
	if( breaker == NULL || breakee == NULL )
		return FALSE;

	if( breaker->item_type != ITEM_WEAPON && breaker->item_type != ITEM_SHIELD && breaker->item_type != ITEM_ARMOR )
		return FALSE;
	
	if( breakee->item_type != ITEM_WEAPON && breakee->item_type != ITEM_SHIELD && breakee->item_type != ITEM_ARMOR )
		return FALSE;
	    
    sprintf( buf, "   %s clashes with %s!", breaker->short_descr, breakee->short_descr );
	act( buf, ch, NULL, NULL, TO_CHAR );
	act( buf, ch, NULL, NULL, TO_ROOM );
	
	sprintf( buf, "   [STR/DU: %d/%d vs. %d/%d, d100(Open)= %d]", breaker->st, breaker->du, 
	breakee->st, breakee->du, roll );
	act( buf, ch, NULL, NULL, TO_CHAR );
	act( buf, ch, NULL, NULL, TO_ROOM );
	
	if( breaker->st >= breakee->st )
	{
		if( roll > breakee->du )
		{
			if( breakee->item_type != ITEM_WEAPON )
			{
			act( "   $p shreds and is rendered utterly useless!", ch, breakee, NULL, TO_CHAR );
			act( "   $p shreds and is rendered utterly useless!", ch, breakee, NULL, TO_ROOM );
			}
			else
			{
			act( "   $p shatters and is rendered utterly useless!", ch, breakee, NULL, TO_CHAR );
			act( "   $p shatters and is rendered utterly useless!", ch, breakee, NULL, TO_ROOM );	
			}
			extract_obj( breakee );
			broke = TRUE;
		}	
	}
	
	if( breakee->st >= breaker->st )
	{
		if( roll > breaker->du )
		{
			if( breaker->item_type != ITEM_WEAPON )
			{
			act( "   $p shreds and is rendered utterly useless!", ch, breaker, NULL, TO_CHAR );
			act( "   $p shreds and is rendered utterly useless!", ch, breaker, NULL, TO_ROOM );
			}
			else
			{
			act( "   $p shatters and is rendered utterly useless!", ch, breaker, NULL, TO_CHAR );
			act( "   $p shatters and is rendered utterly useless!", ch, breaker, NULL, TO_ROOM );
			}
			extract_obj( breaker );
			broke = TRUE;
		}	
	}
	
	return broke;
}



void one_hit( CHAR_DATA *ch, CHAR_DATA *victim, int dt, int ambush )
{

    if ( IS_DEAD(victim) || ch->in_room != victim->in_room )
	return;

	else if( is_safe( ch, victim ) )
	return;

	else
	{
		    int dam;
    int diceroll;
    char buf1[MAX_STRING_LENGTH];
    char buf2[MAX_STRING_LENGTH];
    char buf3[MAX_STRING_LENGTH];
    OBJ_DATA *wield;
    int as = get_melee_as(ch);
    int ds = get_melee_ds(victim);
	int total = 0;
	int adv = 0;
	short bonus = 0;


    wield = get_eq_char( ch, EQUIP_RIGHTHAND );
    
	if ( dt == TYPE_UNDEFINED )
    {
	dt = TYPE_HIT;
	
	if ( wield != NULL && wield->item_type == ITEM_WEAPON )
	    dt += wield->value[0];
    }
	
	//printf("one_hit: %s dt is %d\r\n", ch->name, dt );

    	if( wield == NULL )
		{
		if( IS_NPC( ch ) && IS_SET( ch->act, ACT_BITE ) )
		{
		dt = 1015;
		sprintf( buf1, "$n tries to bite $N!" );
		sprintf( buf2, "$n tries to bite you!" );
		sprintf( buf3, "You try to bite $N!" );
		}
		else if( IS_NPC( ch ) && IS_SET( ch->act, ACT_CHARGE ) )
		{
		dt = 1016;
		sprintf( buf1, "$n charges at $N!" );
		sprintf( buf2, "$n charges at you!" );
		sprintf( buf3, "You charge at $N!" );
		}
		else if( IS_NPC( ch ) && IS_SET( ch->act, ACT_CLAW ) )
		{
		dt = 1017;
		sprintf( buf1, "$n claws at $N!" );
		sprintf( buf2, "$n claws at you!" );
		sprintf( buf3, "You claw at $N!" );
		}
		else if( IS_NPC( ch ) && IS_SET( ch->act, ACT_POUND ) )
		{
		dt = 1018;
		sprintf( buf1, "$n pounds at $N!" );
		sprintf( buf2, "$n pounds at you!" );
		sprintf( buf3, "You pound at $N!" );
		}		
		else
		{
		sprintf( buf1, "$n swings a closed fist at $N!" );
		sprintf( buf2, "$n swings a closed fist at you!" );
		sprintf( buf3, "You swing a closed fist at $N!" );
		}
		
		if( !IS_NPC(ch ) )
		{
		bonus = skill_bonus(ch->pcdata->learned[gsn_brawling]);
		bonus = bonus * ((ch->stance - 100) * -1) / 100;
		as += bonus;
		}
		}
		else
		{
		sprintf( buf1, "$n swings %s at $N!", wield->short_descr );
		sprintf( buf2, "$n swings %s at you!", wield->short_descr );
		sprintf( buf3, "You swing %s at $N!", wield->short_descr );
		
		if( !IS_NPC(ch ) )
		{		
		if( weapon_table[wield->value[0]].weapon_type == WEAPON_TYPE_EDGED )
		bonus = skill_bonus(ch->pcdata->learned[gsn_edged_weapons]);
		else if( weapon_table[wield->value[0]].weapon_type == WEAPON_TYPE_BLUNT )
		bonus = skill_bonus(ch->pcdata->learned[gsn_blunt_weapons]);
		else if( weapon_table[wield->value[0]].weapon_type == WEAPON_TYPE_POLEARM )
		bonus = skill_bonus(ch->pcdata->learned[gsn_polearm_weapons]);
	
		else if( weapon_table[wield->value[0]].weapon_type == WEAPON_TYPE_TWOHANDED  )
		{
		if( get_eq_char( ch, EQUIP_LEFTHAND ) == FALSE )
			bonus = skill_bonus(ch->pcdata->learned[gsn_two_handed_weapons]);
		}
		
		
		bonus = bonus * ((ch->stance - 100) * -1) / 100;
		as += bonus;
		}
		
		}

		act( buf1, ch, NULL, victim, TO_NOTVICT );
		act( buf2, ch, NULL, victim, TO_VICT );
    	act( buf3, ch, NULL, victim, TO_CHAR );

		
		
	diceroll = number_range(1,100);
	
	// fun but too chaotic
	//diceroll = open_1d100();

	// str bonus to as
	if( !IS_NPC(ch) )
		as += stat_bonus( get_curr_Strength( ch ), ch, STAT_STRENGTH );
	
	// weapon enchant bonus to as
	if( wield != NULL && wield->item_type == ITEM_WEAPON 
		&& wield->value[2] > -51 && wield->value[2] < 51)
		as += wield->value[2];
	
	if( !IS_NPC(ch) ) 
	as += ch->pcdata->learned[gsn_combat_maneuvers]/2; // 1 for every 2 ranks of training
	
	
	if( ambush != 0 )
		as += ambush;
	
	// speed bonus to ds
	if( !IS_NPC(victim) )
		ds += stat_bonus(get_curr_Speed( victim ), victim, STAT_SPEED );
		
	// armor enchant bonus to ds
	
	wield = get_eq_char( victim, EQUIP_CHEST );
	if( wield != NULL && wield->item_type == ITEM_ARMOR && 
		wield->value[2] > -51 && wield->value[2] < 51)
		ds += wield->value[2];
	
	// shield enchant bonus to ds

	wield = get_eq_char( victim, EQUIP_LEFTHAND );
	if( wield != NULL && wield->item_type == ITEM_SHIELD && 
		wield->value[2] > -51 && wield->value[2] < 51)
		ds += 20 + wield->value[2];
		
	// shield training bonus to ds
	
	if( !IS_NPC(victim) && skill_bonus(victim->pcdata->learned[gsn_shield_use]) > 0 )
	{
	int shield = 0;
	
	wield = get_eq_char( victim, EQUIP_LEFTHAND );
		
	if( wield != NULL && wield->item_type == ITEM_SHIELD )
	shield = (victim->stance/4) + victim->pcdata->learned[gsn_shield_use]; // Apr '23
	
	if( shield > 0 && victim->stance > 0) // Apr '23
	ds += shield;
	//printf("DS increased by %d due to %s's shield use skill\r\n", shield, victim->name);
	}
	
	// PARRY
	//Base Value = Weapon Ranks + trunc(STR bonus Ã· 4) + trunc(DEX bonus Ã· 4) + (Weapon enchant bonus Ã· 2)
//Parry DS = (Base Value * Stance modifier) + Stance Bonus
	if( !IS_NPC(victim) )
	{
	int weapon = 0;
	
	wield = get_eq_char( victim, EQUIP_RIGHTHAND );
		
	if( wield != NULL && wield->item_type == ITEM_WEAPON )
	{
		int ranks = 0;
		
	// fixme parry bonus needs stance/stat/skill/enchant fixing
	
		if( weapon_table[wield->value[0]].weapon_type == WEAPON_TYPE_EDGED )
		ranks = victim->pcdata->learned[gsn_edged_weapons];
		
		else if( weapon_table[wield->value[0]].weapon_type == WEAPON_TYPE_BLUNT )
		ranks = victim->pcdata->learned[gsn_blunt_weapons];
	
		else if( weapon_table[wield->value[0]].weapon_type == WEAPON_TYPE_POLEARM )
		ranks = victim->pcdata->learned[gsn_polearm_weapons];
	
		else if( weapon_table[wield->value[0]].weapon_type == WEAPON_TYPE_TWOHANDED )
		{
		if( get_eq_char( ch, EQUIP_LEFTHAND ) == FALSE )
		ranks = victim->pcdata->learned[gsn_two_handed_weapons];
		}
		
		else
		ranks = 0;
	
		weapon += ranks;
		
		
	}

	weapon += stat_bonus(get_curr_Speed( victim ), victim, STAT_SPEED );
	weapon += stat_bonus(get_curr_Intelligence( victim ), victim, STAT_INTELLIGENCE ) / 4;
	
	weapon *= (victim->stance + 10);
	weapon /= 100;
	
			ds += weapon;
		//printf("DS increased by %d due to %s's weapon skill\r\n", weapon, victim->name);
	}
	
   // DODGE - fixme armor, shield size, ranged distinction
	
   //Base Value = Dodging Ranks + (AGI Bonus) + trunc(INT Bonus / 4)
   //Evade DS (Melee) = ((Base Value Ã— Armor Hindrance Ã— Shield Factor) - Shield Size Penalty)) Ã— Stance Modifier
   //Evade DS (Ranged) = (Base Value Ã— Armor Hindrance Ã— Shield Factor Ã— Stance Modifier) Ã— 1.5
	
	if( !IS_NPC(victim) && victim->pcdata->learned[gsn_dodging] > 0 )
	{
		int dodge = 0;
		
	dodge += victim->pcdata->learned[gsn_dodging];
	dodge += stat_bonus(get_curr_Speed( victim ), victim, STAT_SPEED );
	dodge += stat_bonus(get_curr_Intelligence ( victim ), victim, STAT_INTELLIGENCE ) / 4;
	
	dodge *= (victim->stance + 10);
	dodge /= 100;
	
				ds += dodge;
		//printf("DS increased by %d due to %s's weapon skill\r\n", dodge, victim->name);
	}
	
	// CM does not provide DS in GS. It never did
	/*
	if( !IS_NPC(victim) && victim->pcdata->learned[gsn_combat_maneuvers] > 2 )
	{
	ds += victim->pcdata->learned[gsn_combat_maneuvers]/2; // regardless of stance +1 ds per 2 ranks
	}
	*/
	
	if( victim->stun > 0 )
	ds -= 20;
	
	// fixme distinguish kneel/sit/prone
	if( victim->position != POS_STANDING )
	ds -= 50;
	
		
	// *** general reminder - have to allow for minuses	
	
	adv += get_adv( ch, victim );
		
	total = as-ds+diceroll+adv;


	sprintf( buf1, "  Att: %s%d - Def: %s%d + Adv. %d + roll: %s%d = %s%d",
	as>-1 ? "+" : "", as, 
	ds>-1 ? "+" : "", ds, 
	adv,
	diceroll>-1 ? "+" : "", diceroll,
	total>-1 ? "+" : "", total );

	act( buf1, ch, NULL, victim, TO_ROOM );
    act( buf1, ch, NULL, victim, TO_CHAR );

/*	
	sprintf( buf1, "%s %s %s.\r\n", combat_string(as), difference_string(as-ds), combat_string2(ds) );

	act( buf1, ch, NULL, victim, TO_NOTVICT );
	act( buf1, ch, NULL, victim, TO_VICT );
    act( buf1, ch, NULL, victim, TO_CHAR );
*/
	
	if( total > 100 )
	{
	dam = (total-100); // df

    if ( dam <= 0 )
	dam = 1;

	//df stuff is handled in damage()
    damage( ch, victim, dam, dt );
	
	if( IS_SET(victim->affected_by, AFF_UNDEAD ) )
	{
		wield = get_eq_char( ch, EQUIP_RIGHTHAND );

		if( wield != NULL && wield->item_type == ITEM_WEAPON )
		{
			if( IS_OBJ_STAT(wield, ITEM_BLESS ) )
			{
				//fixme - this should account for possible multiple affects on an obj.
				if( wield->affected->modifier > 0 )
				wield->affected->modifier--;
	
				if( wield->affected->modifier < 1 )
				{
				act( "$p returns to normal.", ch, wield, NULL, TO_CHAR );
				REMOVE_BIT( wield->extra_flags, ITEM_BLESS );
				wield->affected = NULL;
				}
			}
		}
	}
	
    tail_chain( );
	return;
	}
	else
    {
	/* Miss. */

	//if( total < 101 && total > 0 )
	if( IS_WEAPON(dt) ) // apparently they used a 50% of always missing clash, although I kinda like my initial idea.
	{
		OBJ_DATA *swing = get_eq_char( ch, EQUIP_RIGHTHAND );
		OBJ_DATA *block = get_eq_char( victim, EQUIP_LEFTHAND );
		OBJ_DATA *parry = get_eq_char( victim, EQUIP_RIGHTHAND );
		bool cweapon = FALSE;
		bool shield = FALSE;
		bool vweapon = FALSE;
		
		if( swing != NULL && swing->item_type == ITEM_WEAPON )
			cweapon = TRUE;
		
		if( block != NULL && block->item_type == ITEM_SHIELD )
			shield = TRUE;
		
		if( parry != NULL && parry->item_type == ITEM_WEAPON )
			vweapon = TRUE;
		
		// need the attacker's weapon before it can clash with defender's gear
		if( cweapon == TRUE && number_range(1,100) > 50 )
		{
			if( shield == TRUE && vweapon == TRUE )
			{
			if( number_range(1,100) > 50 )
				breakage_check( ch, swing, victim, parry );
			else
				breakage_check( ch, swing, victim, block );
			}
			else if( vweapon == TRUE )
				breakage_check( ch, swing, victim, parry );
		}
	}	
	
	//printf_to_char ( ch, "Char damage()\r\n" );
	 
	damage( ch, victim, 0, dt );
	
	if( IS_NPC(ch) && number_range(0,100) < 25 )
		wander(ch);

	
	tail_chain( );
	return;
    }

    tail_chain( );
    return;
	
	}
}

char *crit_name( short rank )
{
	switch( rank )
	{
	case 0: return "trifling";
	case 1: return "brushing";
	case 2: return "light";
	
	case 3: return "fair";
	case 4: return "decent";
	
	case 5: return "solid";	
	case 6:	return "strong";
	
	case 7: return "devastating";
	case 8: return "brutal";
	
	case 9: 
	default: return "massive";
	}
}

short get_agidex(CHAR_DATA *ch)
{
short val = stat_bonus(get_curr_Dexterity(ch), ch, STAT_DEXTERITY ) + stat_bonus(get_curr_Speed(ch), ch, STAT_SPEED);

	if( val > 7 && val < 23 )
		return 1;
	else if( val > 22 && val < 38 )
		return 2;
	else if( val > 37 && val < 53 )
		return 3;
	else if( val > 52 && val < 68 )
		return 4;
	else if( val > 67 && val < 83 )
		return 5;
	else if( val > 82 && val < 98 )
		return 6;
	else if( val > 97 && val < 113 )
		return 7;
	else if( val > 112 )
		return 8;
	else
		return 0;
}


/*
 * Inflict damage from a hit.
 */
void damage( CHAR_DATA *ch, CHAR_DATA *victim, int dam, int dt )
{
	int loc = 0;
	int armor = 0;
	int df = 1000;
	int crit = 0;
	bool miss = FALSE;
	bool crit_death = FALSE;
	
	if( dam == 0 )
	miss = TRUE;

    if ( victim->hit < 1 )
	return;

    /*
     * Stop up any residual loopholes.
     */
	 
	 /*
    if ( dam > 1000 )
    {
	bug( "Damage: %d: more than 1000 points!", dam );
	bug( "Dt: %d", dt );
	dam = 1000;
    }
	*/

	
   //if ( victim == ch )
	//return;
	
    if ( victim != ch )
    {
	/*
	 * Certain attacks are forbidden.
	 * Most other attacks are returned.
	 */
	if ( is_safe( ch, victim ) )
	    return;

	/*
	 * More charm stuff.
	 */
	if ( victim->master == ch )
	    stop_follower( victim );
    }
	
	/*
	 * Inviso attacks ... not.
	 */
	 
	/*
	if ( IS_AFFECTED(ch, AFF_INVISIBLE) )
	{
	    affect_strip( ch, gsn_invis );
	    affect_strip( ch, gsn_mass_invis );
	    REMOVE_BIT( ch->affected_by, AFF_INVISIBLE );
	    act( "$n fades into existence.", ch, NULL, NULL, TO_ROOM );
	}
	*/ 
	
	//fixme invis not implemented
	

	// damage factor

	loc = random_hit_location();
	armor = get_armor_hit_loc( victim, loc );
	
	if( IS_BOLT(dt) )
	{
	df = get_bolt_damage_factor( dt, armor );
	
	dam *= df;
	dam /= 1000;
	}
	
	if( IS_MELEE(dt) )
	{
	df = get_damage_factor( dt, armor );
	
	dam *= df;
	dam /= 1000;
	}
	
	if( ( IS_OBJECT(dt) || IS_BOLT(dt) || IS_MELEE(dt) ) && !IS_SET( victim->affected_by, AFF_NONCORP ) )
	{
	crit = dam / get_divisor( armor );
	
	if( crit > MAX_CRIT-1 )
		crit = MAX_CRIT-1;
		
	if( crit < 0 )
		crit = 0;
		
	if( loc > MAX_BODY-1 )
		loc = MAX_BODY-1;
		
	if( loc < 0 )
		loc = 0;

	if( dam > 0 )
	dam += crit_table[loc][crit].damage; 
	}
	
	//printf_to_char( ch, "dt %d    damage %d    crit %d\r\n", dt, dam, crit );
	
	if ( !miss && dam < 1 )
	    dam = 1;
	
	dam_message( ch, victim, dam, dt, loc );

    //fixme damage types should be shown... strike should be burn, slash, puncture, etc..
	
	if( ( IS_OBJECT(dt) || IS_BOLT(dt) || IS_MELEE(dt) ) && dam > 0 && !IS_SET( victim->affected_by, AFF_NONCORP ) )
	{
	char buf[MSL];
	
	if( armor == 0)
	{
	/*	if( crit > 2)
		sprintf( buf, "   %s strike to the %s!", crit_name(crit), body_name[loc] );
		else
		sprintf( buf, "   %s strike to the %s.", crit_name(crit), body_name[loc] ); */	
		;
	}
	else
	{
		OBJ_DATA *weapon = get_eq_char( ch, EQUIP_RIGHTHAND );
		//OBJ_DATA *armor_obj = get_armor_hit_obj( victim, loc ); 
	
	/*
		if( crit > 2)
		sprintf( buf, "   The %s strike penetrates the armor covering the %s!", crit_name(crit), body_name[loc] );
		else
		sprintf( buf, "   The strike is mostly absorbed by the armor covering the %s.", body_name[loc] );
	*/
	
		if( IS_WEAPON(dt) && weapon != NULL )
		{	
			if( get_armor_hit_obj( victim, loc ) != NULL )
			{		
				breakage_check( ch, weapon, victim, get_armor_hit_obj( victim, loc ) );
			}
		}
	
	}
	
	/*
	act( buf, ch, NULL, victim, TO_NOTVICT );
    act( buf, ch, NULL, victim, TO_CHAR );
    act( buf, ch, NULL, victim, TO_VICT );	
	*/
	
	act( crit_table[loc][crit].msg, ch, NULL, victim, TO_NOTVICT );
    act( crit_table[loc][crit].msg, ch, NULL, victim, TO_CHAR );
    act( crit_table[loc][crit].msg, ch, NULL, victim, TO_VICT );	
	}
	
	//  wake up
	if ( IS_AFFECTED(victim, AFF_SLEEP) || is_affected( victim, gsn_sleep ) )
	{
	affect_strip( victim, gsn_sleep );
	REMOVE_BIT( victim->affected_by, AFF_SLEEP );
	}
	
		// get mad
		
	
	if( IS_NPC(victim) && dt != TYPE_BLEED )
		victim->hunting = ch;
	

    /*
     * Hurt the victim.
     */
    victim->hit -= dam;
    
	
	//gods can die
	/*
	if ( !IS_NPC(victim)
    &&   victim->level >= LEVEL_IMMORTAL
    &&   victim->hit < 1 )
	victim->hit = 1;	
	*/

	if( dt != TYPE_BLEED )
	{

	if( crit > 0 && !IS_SET( victim->affected_by, AFF_NONCORP ) )
	{		
		if( IS_SET( crit_table[loc][crit].status, STATUS_DEATH ) )
		{
/*
		act( "   The critical hit causes $N's death.", ch, NULL, victim, TO_NOTVICT );
		act( "   The critical hit causes $N's death.", ch, NULL, victim, TO_CHAR );
		act( "   The critical hit causes your death.", ch, NULL, victim, TO_VICT ); 
*/	
			crit_death = TRUE;
		}
		
		else if( victim->stun < 1 && victim->hit > 1 )
		{
			act( "   $N is stunned!", ch, NULL, victim, TO_NOTVICT );
			act( "   $N is stunned!", ch, NULL, victim, TO_CHAR );
			act( "   You are stunned!", ch, NULL, victim, TO_VICT );
		
		if( IS_SET( crit_table[loc][crit].status, STATUS_STUN_1 ) )
			add_stun( victim, 1*5 );
		
		if( IS_SET( crit_table[loc][crit].status, STATUS_STUN_2 ) )
			add_stun( victim, 2*5 );
		
		if( IS_SET( crit_table[loc][crit].status, STATUS_STUN_3 ) )
			add_stun( victim, 3*5 );
		
		if( IS_SET( crit_table[loc][crit].status, STATUS_STUN_4 ) )
			add_stun( victim, 4*5 );
		
		if( IS_SET( crit_table[loc][crit].status, STATUS_STUN_5 ) )
			add_stun( victim, 5*5 );
		
		if( IS_SET( crit_table[loc][crit].status, STATUS_STUN_6 ) )
			add_stun( victim, 6*5 );
		
		if( IS_SET( crit_table[loc][crit].status, STATUS_STUN_7 ) )
			add_stun( victim, 7*5 );
		
		if( IS_SET( crit_table[loc][crit].status, STATUS_STUN_8 ) )
			add_stun( victim, 8*5 );
		
		if( IS_SET( crit_table[loc][crit].status, STATUS_STUN_9 ) )
			add_stun( victim, 9*5 );
		
		if( IS_SET( crit_table[loc][crit].status, STATUS_STUN_10 ) )
			add_stun( victim, 10*5 );

		if( victim->position != POS_PRONE && IS_SET( crit_table[loc][crit].status, STATUS_KNOCKDOWN ) )
		{
			act( "   $N is knocked down to the ground!", ch, NULL, victim, TO_NOTVICT );
			act( "   $N is knocked down to the ground!", ch, NULL, victim, TO_CHAR );
			act( "   You are knocked down to the ground!", ch, NULL, victim, TO_VICT );
			victim->position = POS_PRONE;
		}
		}
		
		if( IS_DEAD(victim) || crit_death == TRUE )  // situations where they are going to die, we don't want...
		wound( victim, loc, crit_table[loc][crit].wound, crit_table[loc][crit].bleed, TRUE );    // to display messages re bleeding/KD
		else
		wound( victim, loc, crit_table[loc][crit].wound, crit_table[loc][crit].bleed, FALSE );	

		//check for flares, but only if the victim is still alive
		if( dt >= TYPE_HIT && !IS_DEAD(victim) )
		{
			int x;
			OBJ_DATA *wield = get_eq_char( ch, EQUIP_RIGHTHAND );
		
			if( wield != NULL && wield->value[3] == WEAPON_FIRE_FLARES && number_range(1,5) == 5)
			{
			x = number_range(1,10);
			act( " ** Your $o emits a rush of fire! **", ch, wield, NULL, TO_CHAR );
			act( " ** $n's $o emits a rush of fire! **", ch, wield, NULL, TO_ROOM );
			damage( ch, victim, x*5, DAMAGE_TYPE_FIRE );
			}
		}
	}
}	

/*
	if( victim->hit < 1 && crit_death == FALSE )
	{ 
		if( IS_SET( victim->affected_by, AFF_NONCORP ) || IS_SET( victim->affected_by, AFF_UNDEAD ) )
		{
		act( "   $N has been destroyed.", ch, NULL, victim, TO_NOTVICT );
		act( "   $N has been destroyed.", ch, NULL, victim, TO_CHAR );
		act( "  $N has been destroyed.", ch, NULL, victim, TO_VICT ); 
		}
		else
		{
		act( "   $N bleeds out.", ch, NULL, victim, TO_NOTVICT );
		act( "   $N bleeds out.", ch, NULL, victim, TO_CHAR );
		act( "   You bleed out.", ch, NULL, victim, TO_VICT ); 
		}
	}
*/
	
    /*
     * Payoff for killing things.  	group_gain( ch, victim ) should be moved to when really_kill fires, but we're not passing ch, victim yet...
     */
    if ( victim->hit < 1 || ( crit_death == TRUE && !IS_IMMORTAL(victim) ) )
    {
	group_gain( ch, victim );

	if ( !IS_NPC(victim) )
	{
	    sprintf( log_buf, "%s killed by %s at %d",
		victim->name,
		(IS_NPC(ch) ? ch->short_descr : ch->name),
		victim->in_room->vnum );
	    log_string( log_buf );

	    /*
	     * Dying penalty:
	     * 10K experience lost.  Ouch.
	     */
		 
		sprintf( log_buf, "\r\n   * %s is no more.\r\n", victim->name);
		
		do_echo( victim, log_buf );
		
		if( victim->level > 1 )
		gain_exp( victim, -10000 );
		
		//mercy - fixme deeds
		if( victim->exp < 0 )
			victim->exp = 0;
		
		free_string( log_buf );
	}
	
	//fixme - if outlaw/noquit implemented, remove it here if appropriate. reward bounties

	// killing does indeed sate the hunter.
	if( ch->hunting != NULL && ch->hunting == victim )
		ch->hunting = NULL;

	raw_kill( victim );
    }
	
	if( IS_MELEE(dt) )
	{
	int rt = weapon_table[dt-TYPE_HIT].weapon_speed;
	
	//fixme fix rt - make it worse for plate, chain groups?
	
	if( !IS_NPC( ch ) )
	{
	short val = 0;
	
	
	OBJ_DATA *armor_obj = get_eq_char( ch, EQUIP_CHEST );
	
	if( armor_obj != NULL && armor_obj->item_type == ITEM_ARMOR 
		&& armor_obj->value[3] > -1 && armor_obj->value[3] < MAX_ARMOR )
	{
	if( armor_table[armor_obj->value[3]].RT > 0 )
	{
		val = armor_table[armor_obj->value[3]].RT;
		
		val -= skill_bonus( ch->pcdata->learned[gsn_armor_use] ) / 10;
		
		if( val > 0 )
			rt += val;
	}
	}
	
	}
	
	if( !IS_NPC(ch) && get_encumbrance_level(ch) )
	{
		rt += get_encumbrance_level(ch) / 10;
	}
	
	if( !IS_NPC(ch) && rt > weapon_table[dt-TYPE_HIT].weapon_speed )
		rt -= get_agidex( ch );
	
	if( rt < weapon_table[dt-TYPE_HIT].weapon_speed )
	rt = weapon_table[dt-TYPE_HIT].weapon_speed;
	
	if( rt > 15 )
		rt = 15;
	
	add_roundtime( ch, rt );

    show_roundtime( ch, rt );
	}
	
	
    tail_chain( );
    return;
}



bool is_safe( CHAR_DATA *ch, CHAR_DATA *victim )
{
	if ( IS_DEAD(victim) )
	{
		send_to_char("They are already dead.\r\n", ch );
		return TRUE;
	}
	
	// no law for gods
	if( !IS_IMMORTAL(ch ) )
	{
	// no PvP for level 1s killees.
	if( !IS_NPC(victim) && !IS_NPC( ch ) && victim->level < 2 )
	{
	send_to_char("You cannot attack other players under level 2.\n\r", ch );
	return TRUE;
    }
	
	// no PvP for level 1s killers.
	if( !IS_NPC(ch) && !IS_NPC( victim ) && ch->level < 2 )
	{
	send_to_char("You cannot attack other players until you reach level 2.\n\r", ch );
	return TRUE;
    }
	
	// please don't disturb the scenery
	if ( !IS_NPC(ch) && IS_NPC(victim) && IS_SET( victim->act, ACT_TOWN )
			&& ( is_name( ch->in_room->area->name, "Scunthorpe" ) || is_name( ch->in_room->area->name, "Reng")  ) )
    {
	printf_to_char( ch, "As you prepare to attack %s, you realize that doing so would mean a warrant for your arrest.  Perhaps this crime can wait for another day.\n\r", victim->short_descr );
	return TRUE;
    }
	
	if ( IS_SET( ch->in_room->room_flags, ROOM_SAFE ) )
    {
	send_to_char("Be at peace, there is no need for combat here.\n\r", ch );
	return TRUE;
    }
	}
	
    return FALSE;
}






// dump all loot to room
void make_corpse( CHAR_DATA *ch )
{
    OBJ_DATA *obj;
    OBJ_DATA *obj_next;

	if ( ch->gold > 0 )
	{
		OBJ_DATA *coin;
		OBJ_DATA *coin_next;
		int gold = ch->gold;
		
		for ( coin = ch->in_room->contents; coin != NULL; coin = coin_next )
		{
	    coin_next = coin->next_content;

	    switch ( coin->pIndexData->vnum )
	    {
	    case OBJ_VNUM_MONEY_ONE:
		gold += 1;
		extract_obj( coin );
		break;

	    case OBJ_VNUM_MONEY_SOME:
		gold += coin->value[0];
		extract_obj( coin );
		break;
	    }
		}
	
		obj_to_room( create_money(gold), ch->in_room );
		ch->gold = 0;		
	}

    for ( obj = ch->carrying; obj != NULL; obj = obj_next )
    {
	obj_next = obj->next_content;
	obj_from_char( obj );
	obj_to_room( obj, ch->in_room );
	}

    return;
}

void permadeath( CHAR_DATA * ch )
{
   char strsave[MAX_INPUT_LENGTH];
   DESCRIPTOR_DATA *d, *d_next;
 
   if ( IS_NPC( ch ) || IS_IMMORTAL( ch ) )
      return;
 
   send_to_char( "Your soul vanishes from the land forever.\n\r", ch );
   
   //fixme world message
   
   sprintf( strsave, "%s%s", PLAYER_DIR, capitalize( ch->name ) );
 
   sprintf( log_buf, "\r\n   * %s's soul has been lost to the darkness.\r\n", ch->name);
   do_echo( ch, log_buf );
 
   sprintf( log_buf, "%s has been deleted!", ch->name );
   log_string( log_buf );
   unlink( strsave );
 
   d = ch->desc;
   extract_char( ch, TRUE );
   if ( d != NULL )
      close_socket( d );
 
   for ( d = descriptor_list; d != NULL; d = d_next )
   {
      CHAR_DATA *tch;
      d_next = d->next;
      tch = d->original ? d->original : d->character;
      if ( tch )
      {
         extract_char( tch, TRUE );
         close_socket( d );
      }
   }
   return;
}
 

void deed_warning( CHAR_DATA *ch )
{
	
	printf_to_char( ch, "It seems you have perished.  Although you cannot move, you remain totally aware of what is going on around you...\r\n\r\n" );


	if( ch->pcdata->deeds == 0 )
	{
	printf_to_char( ch, "You have no outstanding favors which the gods owe you, and without having accomplished a deed for them, your soul is in danger of vanishing from the land forever!\r\n" );
	}
	else
	{
	printf_to_char( ch, "You are relieved when you remember that the gods owe you a favor.\r\n" );	
	}
	
}

void death_cry( CHAR_DATA *ch )
{
	if ( IS_SET( ch->affected_by, AFF_NONCORP ) )
	{
	//fixme intelligent vs animal vs undead vs noncorp death
	act( "$n starts to rapidly dissipate and spiral into nothingness!", ch, NULL, NULL, TO_ROOM );
    act( "You start to rapidly dissipate and spiral into nothingness!", ch, NULL, NULL, TO_CHAR );
	}
	else
	{

	//fixme intelligent vs animal vs undead vs noncorp death
	if( ch->position == POS_STANDING )
	{
	ch->position = POS_PRONE;	
	act( "$n collapses to the ground, dead.", ch, NULL, NULL, TO_ROOM );
    act( "You colllapse to the ground, dead.", ch, NULL, NULL, TO_CHAR );
	}
	else
	{
	if ( IS_SET( ch->act, ACT_ANIMAL ) )
	{
	act( "$n twitches violently in $s death throes.", ch, NULL, NULL, TO_ROOM );
    act( "You twitch violently in your death throes.", ch, NULL, NULL, TO_CHAR );
	}
	else
	{
	act( "$n lets out one final scream and dies.", ch, NULL, NULL, TO_ROOM );
    act( "You let out one final scream and die.", ch, NULL, NULL, TO_CHAR );
	}	
	}
	
	if(!IS_NPC(ch))
		deed_warning(ch);
	
	}
    return;
}



void raw_kill( CHAR_DATA *victim )
{
	if( !IS_NPC( victim ) && victim->level == 1 )
	{
		int iBody = 0;
		
		printf_to_char( victim, "*** Because you are still %d experience from level 2, you are being rescued automatically.\r\n\r\n*** You have %d deeds.  Decaying or DEPARTing with no deeds will result in permadeath.\r\n\r\n", 10000-victim->exp, victim->pcdata->deeds );
		
		victim->position = POS_PRONE;
		
		while( iBody < MAX_BODY )
		{
		if( victim->body != 0 )
		{
		victim->bandage[iBody] = 0;
		victim->bleed[iBody] = 0;
		victim->body[iBody] = 0;
		victim->scars[iBody] = 0;
		}
		
		iBody++;
		}
	
		victim->hit = 1;
		victim->mana = 0;
		victim->move = 1;
		victim->stun = 0;
		victim->wait = 0;
		victim->pcdata->fieldExp = 0;
		
	
		char_from_room( victim );
		char_to_room( victim, get_room_index( ROOM_VNUM_TEMPLE ) );
		
		printf_to_char(  victim, "Suddenly, you wake up in a beautifully carved white marble temple.\r\n" );	
		act( "$n suddenly appears in the center of the temple.", victim, NULL, NULL, TO_ROOM );
		
		send_to_char( "You are somehow alive again, but cannot yet move as pain wracks your body.\r\n", victim );
		add_roundtime( victim, 60 );
		add_stun( victim, 5 );
		return;
	}
	
    death_cry( victim );
	
	victim->stun = 0;
	victim->wait = 0;
	
	//droppage
	
	if( !IS_NPC( victim ) )
	{
	drop_hand( victim, EQUIP_RIGHTHAND );
	drop_hand( victim, EQUIP_LEFTHAND );
	
	victim->pcdata->fieldExp = 0;
	}
	
	SET_BIT( victim->affected_by, AFF_DEAD );
	
	if( !IS_NPC( victim ) )
	victim->decay = 600; //fixme racial?
	else
	{
	if( IS_SET( victim->affected_by, AFF_NONCORP ) )
	victim->decay = number_range(10,20);
	else
	victim->decay = number_range(30,40);
	}
	
    return;
}

void do_loot ( CHAR_DATA *ch, char *argument )
{
	if( argument[0] != '\0' )
	{
	do_search( ch, argument );
	return;
	}

	send_to_char( "Loot what?\r\n", ch); // fixme no empty looting yet
	return;

}

//fixme search first available corpse
void do_search( CHAR_DATA *ch, char *argument )
{
    CHAR_DATA *victim;
	
    if ( argument[0] == '\0' )
    {
	//send_to_char( "Search what?\r\n", ch );
	do_search_room( ch, argument ); // call to search room to search for hidden things...
	return;
    }
	
	if ( ( victim = get_char_room( ch, argument ) ) == NULL )
	{
	    send_to_char( "They aren't here.\r\n", ch );
	    return;
	}
	
	if( !IS_NPC( victim ) )
	{
	    send_to_char( "You will have to wait for them to rot away.\r\n", ch );
	    return;
	}	
	
	if( !IS_SET( victim->affected_by, AFF_DEAD ) )
	{
	    send_to_char( "You can't search them! They're not dead yet!\r\n", ch );
	    return;
	}
	
	act( "$n searches you, looking for valuables.", ch, NULL, victim, TO_VICT );
	act( "$n searches $N, looking for valuables.", ch, NULL, victim, TO_NOTVICT );
    act( "You search $N, looking for valuables.", ch, NULL, victim, TO_CHAR );
	
	if( !IS_SET(victim->act, ACT_ANIMAL ) )
	victim->gold += dice(victim->level,5);
	
	if( victim->gold > 0 )
	{
	if( !IS_NPC( ch ) && IS_SET( ch->act, PLR_COINS ) )
	{
	printf_to_char( ch, "You find %d silver coin%s that you quickly pocket.\r\n", victim->gold, victim->gold == 1 ? "" : "s" );
	ch->gold = ch->gold + victim->gold;
	victim->gold = 0;
	}
	else
	printf_to_char( ch, "You find %d silver coin%s.\r\n", victim->gold, victim->gold == 1 ? "" : "s" );
	}
	
	//fixme generate treasure

	
	
	if( !IS_SET( victim->act, ACT_ANIMAL ) )
	{
	
	if( number_range(1,7) < 3)
	{
	if ( number_range(1,20) == 20)
	obj_to_char( create_random_magic_item( number_range(0, MAX_MAGIC_ITEM-1 )), victim  );
	else
	obj_to_char( create_random_gem( victim->level ), victim );
	}
	
	if ( number_range(1,20) == 20 )
	obj_to_char( create_random_box( victim->level ), victim );	

	if( number_range(1,20) == 20 )
	obj_to_char( create_random_container( victim->level ), victim );	

	if ( number_range(1,75) == 75 )
	obj_to_char( create_object( get_obj_index( OBJ_VNUM_LOCKPICK ), 0 ), victim );		

	//old school generation once in a while.
	if ( number_range(1,100) == 100 )
	{
	if( number_range(1,9) < 4)
	{
	if ( number_range(1,9) < 6) 
	obj_to_char( create_random_armor( victim->level ), victim );
	else
	obj_to_char( create_random_weapon( victim->level ), victim );
	}
	}

	}
	
	new2_show_list_to_char( victim->carrying, ch, TRUE, FALSE, "You find " );

	really_kill( victim, TRUE );
}


void really_kill( CHAR_DATA *victim, bool dropStuff )
{
	int iBody = 0;
	
	if( dropStuff == TRUE )
    make_corpse( victim );
	
	
	if( IS_SET( victim->affected_by, AFF_NONCORP ) )
	{
	act( "$n fades into nothingness as $e dematerializes.", victim, NULL, NULL, TO_ROOM );
    act( "You fade into nothingness as you dematerialize.", victim, NULL, NULL, TO_CHAR );
	}
	else
	{
	act( "$n quickly decomposes before your eyes.", victim, NULL, NULL, TO_ROOM );
    act( "You quickly decompose.", victim, NULL, NULL, TO_CHAR );
	}

    //worn objects go to ground. fixme
	
    if ( IS_NPC(victim) )
    {
	victim->pIndexData->killed++;
	kill_table[URANGE(0, victim->level, MAX_LEVEL-1)].killed++;
	extract_char( victim, TRUE );
	return;
    }
	
	extract_char( victim, FALSE );
	
    while ( victim->affected )
	affect_remove( victim, victim->affected );
	
    victim->affected_by	= 0;
    victim->position	= POS_PRONE;
    victim->hit		= 1;
    victim->mana	= 1;
    victim->move	= 1;
	
	if( !IS_NPC( victim ) )
	{
		
	if( victim->pcdata->deeds < 1 )
	{
		permadeath(victim);
		return;
	}
	
	else
	{
	victim->pcdata->deeds--;
		
    save_char_obj( victim );
	
	//send_to_char( VT100_ERASE_SCREEN, victim );
	
	printf_to_char( victim, "Your surroundings blur and fade away.\r\n\r\n");
	
	//fixme formatting 
	printf_to_char( victim, "You find yourself lost and wandering on a plane of darkness.\r\n" );

	printf_to_char( victim, "You are surrounded by storms of shadows.  You feel numb as a pinprick of light on the far horizon beckoning to you.\r\n\r\n" );
	
	printf_to_char( victim, "Time loses meaning as you move slowly toward the distant point of light while hopelessness sets in. It seems so far away, and you feel colder as your progress grow slower.\r\n");
	
	printf_to_char( victim, "Your sojourn could have taken a minute or a lifetime, but you finally reach the point of light, almost as your will was starting to break, but then the light encompasses you totally...\r\n");

	printf_to_char( victim, "The brightness of the divine light bathes you in a warmth which increases in intensity and you feel exquisite pain as your body is reborn, bit by bit.\r\n\r\n");
					
	printf_to_char( victim, "You awake in a beautifully carved white marble temple, naked and in a cold sweat.\r\n");

	act( "$n suddenly appears in the center of the temple, $e is naked and sweating.", victim, NULL, NULL, TO_ROOM );
	
	while( iBody < MAX_BODY )
	{
		if( victim->body != 0 )
		{
		victim->bandage[iBody] = 0;
		victim->bleed[iBody] = 0;
		victim->body[iBody] = 0;
		victim->scars[iBody] = 0;
		}
		
	iBody++;
	}
	
		//10 round stun on temple altar. maybe i'll put this back one day.
		//add_stun( victim, 10*5 );
		return;
	}
	}
}


//no way to group anyone currently.
//fixme should only get credit if you DAMAGED a creature significantly
void group_gain( CHAR_DATA *ch, CHAR_DATA *victim )
{
    CHAR_DATA *gch;
    int xp;
    int members;

    /*
     * Monsters don't get kill xp's or alignment changes.
     * P-killing doesn't help either.
     * Dying of mortal wounds or poison doesn't give xp to anyone!
     */
    if ( IS_NPC(ch) || !IS_NPC(victim) || victim == ch )
	return;

    members = 0;
    for ( gch = ch->in_room->people; gch != NULL; gch = gch->next_in_room )
    {
	if ( is_same_group( gch, ch ) )
	    members++;
    }

    if ( members == 0 )
    {
	bug( "Group_gain: members.", members );
	members = 1;
    }

    for ( gch = ch->in_room->people; gch != NULL; gch = gch->next_in_room )
    {
	if ( !is_same_group( gch, ch ) )
	    continue;

	xp = xp_compute( gch, victim ) / members;
	gain_exp( gch, xp );
	}
	
    return;
}



// no gimping
int xp_compute( CHAR_DATA *gch, CHAR_DATA *victim )
{
	int xp = 100 + 10 * (victim->level - gch->level);

	if( xp > 150 )
		xp = 150;

	if( xp < 0 )
		xp = 0;

    return xp;
}



void dam_message( CHAR_DATA *ch, CHAR_DATA *victim, int dam, int dt, int loc )
{
	char buf[MAX_STRING_LENGTH];
	
	if( dt == TYPE_BLEED )
	return;

    if( dam < 1 )
    {
	sprintf( buf, "   A clean miss." );
	}
    else
    {
	sprintf( buf, "   ... and the %s (DT %d) hits the %s (covered in %s) for %d points of damage!",  
		dt_to_name(dt), dt,
		body_name[loc], 
		ag_to_name( get_armor_hit_loc(victim, loc) ),
		dam );

	if( IS_MELEE(dt) )
	sprintf( buf, "   ... and hits for %d points of damage!", dam );
	else
	sprintf( buf, "   ... %d points of damage!", dam );
    }

	if( !IS_BLEED(dt) )
	{
    act( buf, ch, NULL, victim, TO_NOTVICT );
    act( buf, ch, NULL, victim, TO_CHAR );
    act( buf, ch, NULL, victim, TO_VICT );
	}
	
	//free_string( buf );
	
    return;
}



void disarm( CHAR_DATA *ch, CHAR_DATA *victim )
{
}




void trip( CHAR_DATA *ch, CHAR_DATA *victim )
{
}



void do_kill( CHAR_DATA *ch, char *argument )
{
    char arg[MAX_INPUT_LENGTH];
    CHAR_DATA *victim;

    one_argument( argument, arg );

    if ( arg[0] == '\0' )
    {
	send_to_char( "Kill whom?\r\n", ch );
	return;
    }

    if ( ( victim = get_char_room( ch, arg ) ) == NULL )
    {
	send_to_char( "They aren't here.\r\n", ch );
	return;
    }

    if ( victim == ch )
    {
	send_to_char( "Kill yourself?\r\n", ch );
	return;
    }

    if ( is_safe( ch, victim ) )
	return;

    if ( IS_AFFECTED(ch, AFF_CHARM) && ch->master == victim )
    {
	act( "$N is your beloved master.", ch, NULL, victim, TO_CHAR );
	return;
    }
	
	if( !IS_NPC(ch) )
	{
		OBJ_DATA *weapon = get_eq_char( ch, EQUIP_RIGHTHAND );
		
		if( ch->pcdata->learned[gsn_brawling] < 1 )
		{
		if( weapon && weapon->item_type != ITEM_WEAPON )
		{
			send_to_char( "But you know nothing of brawling.  Perhaps get a weapon?\r\n", ch );
			return;
		}
		}
		
		
		// need a blessed weapon for undead
		if( IS_SET(victim->affected_by, AFF_UNDEAD ) 
	   && (weapon == NULL || 
      ( weapon && !IS_OBJ_STAT(weapon, ITEM_BLESS ) 
	    ) ) )
		{
		send_to_char( "You attack but quickly realize your mundane weapon has no effect!\r\n", ch );
		act( "$n's attack on $N has no effect!", ch, NULL, victim, TO_ROOM );
		
		add_roundtime( ch, 3 );
		show_roundtime( ch, 3 );
		return;
		}
	}
	

    multi_hit( ch, victim, TYPE_UNDEFINED );
    return;
}



void do_sla( CHAR_DATA *ch, char *argument )
{
    send_to_char( "If you want to SLAY, spell it out.\r\n", ch );
    return;
}


void do_slay( CHAR_DATA *ch, char *argument )
{
    CHAR_DATA *victim;
    char arg[MAX_INPUT_LENGTH];

    one_argument( argument, arg );
	
	//fixme
	if( !str_cmp( arg, "all" ) )
	{
	send_to_char( "Not implemented.\r\n", ch );
	return;
	}

    if ( ( victim = get_char_room( ch, arg ) ) == NULL )
    {
	send_to_char( "They aren't here.\r\n", ch );
	return;
    }

    if ( ch == victim )
    {
	send_to_char( "Slay yourself?\r\n", ch );
	return;
    }
	

    if ( !IS_NPC(victim) && victim->level >= ch->level )
    {
	send_to_char( "Nope, too high level.\r\n", ch );
	return;
    }

    raw_kill( victim );
    
    return;
}

void do_depart( CHAR_DATA *ch, char *argument )
{
	if ( !IS_SET( ch->affected_by, AFF_DEAD ) )
	{
		send_to_char("You are not dead.  Yet.\r\n", ch );
		return;
	}
	
	if( str_cmp( argument, "confirm" ) )
	{
		send_to_char("If you are sure you wish to depart and continue on your journey, DEPART CONFIRM.\r\n", ch );
		
		if( !IS_NPC(ch) && ch->pcdata->deeds < 1 )
		{
			send_to_char("* WARNING:  With no deeds, your soul will vanish from the land forever!\r\n", ch );
		}
		return;
	}
	
	really_kill( ch, TRUE );
}

void do_stance( CHAR_DATA *ch, char *argument )
{
	if( is_name( argument, "defensive" ) )
		ch->stance = 100;
	else if( is_name( argument, "guarded" ) )
		ch->stance = 80;
	else if( is_name( argument, "neutral" ) )
		ch->stance = 60;
	else if( is_name( argument, "forward" ) )
		ch->stance = 40;
	else if( is_name( argument, "advanced" ) )
		ch->stance = 20;
	else if( is_name( argument, "offensive" ) )
		ch->stance = 0;
	
	switch( ch->stance )
	{
	case 100: send_to_char( "You are now in a defensive stance.\r\n", ch ); break;
	case 80: send_to_char( "You are now in a guarded stance.\r\n", ch ); break;
	case 60: send_to_char( "You are now in a neutral stance.\r\n", ch ); break;
	case 40: send_to_char( "You are now in a forward stance.\r\n", ch ); break;
	case 20: send_to_char( "You are now in an advanced stance.\r\n", ch ); break;
	case 0: send_to_char( "You are now in an offensive stance.\r\n", ch ); break;
	default: send_to_char( "error: bad stance.\r\n", ch ); break;
	}
	
}

void do_tend( CHAR_DATA *ch, char *argument )
{
    char arg1 [MAX_INPUT_LENGTH];
    char arg2 [MAX_INPUT_LENGTH];
    CHAR_DATA *victim;
	int skill = 0;
	bool self = FALSE;
	bool found = FALSE;
	short i;

    argument = one_argument( argument, arg1 );
    argument = one_argument( argument, arg2 );
	
	if ( arg1[0] == '\0' || arg2[0] == '\0' )
    {
	send_to_char( "Tend whom where?\n\r", ch );
	return;
    }

    if ( ( victim = get_char_room( ch, arg1 ) ) == NULL )
    {
	send_to_char( "They aren't here.\n\r", ch );
	return;
    }

    if ( victim == ch )
		self = TRUE;
	
	if(!IS_NPC(ch))
		skill = ch->level * 2; //intelligent NPCs will eventually be able to tend fixme
	else
		skill = skill_bonus( ch->pcdata->learned[gsn_first_aid] );
	
	if( skill == 0 )
	{
		send_to_char( "You don't know the first thing about first aid.\r\n", ch );
		return;
	}
	
	if( victim->position != POS_PRONE )
	{
		printf_to_char(ch, "%s should lay down first.\r\n", capitalize(HE_SHE(victim)) );
		return;
	}
	
	for( i = 0; i < MAX_BODY; i++ )
	{
		if( is_name( arg2, body_name[i] ) && victim->bleed[i] > 0 )
		{
			found = TRUE;
			break;
		}
	}
	
	if( found  )
	{
		if( victim->bleed[i] * 10 > skill )
		{
			send_to_char( "The severity of that wound is too great for your skills.\r\n", ch );
			return;
		}
		
	if( self )
	{
		printf_to_char(ch, "You attempt to bandage your %s.\r\n", body_name[i] );
		act( "$n starts bandaging $mself.", ch, NULL, NULL, TO_ROOM );
		victim->bandage[i] = skill / 10;
		
			if( victim->bandage[i] < 1 )
			victim->bandage[i] = 0;
		return;
	}
	else
	{
		printf_to_char(ch, "You attempt to bandage the %s.\r\n", body_name[i] );
		act( "$n starts bandaging $N.", ch, NULL, victim, TO_NOTVICT );
		act( "$n starts bandaging you.", ch, NULL, victim, TO_VICT );
		victim->bandage[i] = skill / 10;
		
		if( victim->bandage[i] < 1 )
			victim->bandage[i] = 0;
		return;
	}
	}
	else
	{
		send_to_char( "That area isn't bleeding.\r\n", ch );
		return;
	}
	
	return;
}



#if defined(macintosh)
#include <types.h>
#else
#include <sys/types.h>
#endif
#include <ctype.h>
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <time.h>
#include "merc.h"

#if !defined(macintosh)
extern	int	_filbuf		args( (FILE *) );
#endif



extern char str_empty[1];
extern char *top_string;
extern char *string_space;
extern int nAllocString;
extern int sAllocString;
extern bool fBootDb;
extern FILE *fpArea;
extern char strArea[MAX_INPUT_LENGTH];

extern char *			string_hash		[MAX_KEY_HASH];

/*
 * Read a letter from a file.
 */
char fread_letter( FILE *fp )
{
    char c;

    do
    {
	c = getc( fp );
	//printf("%c"); //debug only fixme
    }
    while ( isspace(c) );

    return c;
}



/*
 * Read a number from a file.
 */
int fread_number( FILE *fp )
{
    int number;
    bool sign;
    char c;

    do
    {
	c = getc( fp );
	//printf("%c",c); //fixme debug only
    }
    while ( isspace(c) );

    number = 0;

    sign   = FALSE;
    if ( c == '+' )
    {
	c = getc( fp );
    }
    else if ( c == '-' )
    {
	sign = TRUE;
	c = getc( fp );
    }

    if ( !isdigit(c) )
    {
	bug( "Fread_number: bad format.", 0 );
	printf("%c found when expecting digit", c); //fixme
	exit( 1 );
    }

    while ( isdigit(c) )
    {
	number = number * 10 + c - '0';
	c      = getc( fp );
    }

    if ( sign )
	number = 0 - number;

    if ( c == '|' )
	number += fread_number( fp );
    else if ( c != ' ' )
	ungetc( c, fp );

    return number;
}


/*
 * Read and allocate space for a string from a file.
 * These strings are read-only and shared.
 * Strings are hashed:
 *   each string prepended with hash pointer to prev string,
 *   hash code is simply the string length.
 * This function takes 40% to 50% of boot-up time.
 */
char *fread_string( FILE *fp )
{
    return fread_string_full( fp, '~', TRUE );
}

/*
 * This function allows leading whitespace.
 */
char *fread_description( FILE *fp )
{
    return fread_string_full( fp, '~', FALSE );
}

/*
 * This allows strings terminated by newline and tilde only.
 */
char *fread_string_full( FILE *fp, char terminator, bool fKillSpace )
{
    char *plast;
    char c;

    plast = top_string + sizeof(char *);
    if ( plast > &string_space[MAX_STRING - MAX_STRING_LENGTH] )
    {
	bug( "Fread_string: MAX_STRING %d exceeded.", MAX_STRING );
	exit( 1 );
    }

    /*
     * Skip blanks.
     * Read first char.
     */
    if ( fKillSpace )
    {
      do
      {
	c = getc( fp );
      }
      while ( isspace(c) );
    }
    else
    {
      do
      {
        c = getc( fp );
      }
      while( c == '\n' || c == '\r' );
    }

    if ( ( *plast++ = c ) == '~' )
	return &str_empty[0];

    for ( ;; )
    {
	/*
	 * Back off the char type lookup,
	 *   it was too dirty for portability.
	 *   -- Furey
	 */
	switch ( *plast = getc( fp ) )
	{
	default:
	    plast++;
	    break;

	case EOF:
	    bug( "Fread_string: EOF", 0 );
	    exit( 1 );
	    break;

	case '\\':
	    plast++;
	    *plast = getc( fp );
	    if ( *plast == '~' )
		plast[-1] = '~';
	    else
		plast++;
	    break;

	case '\r':
	    break;

	case '\n':
	    if ( terminator != '\n' )
	    {
		plast++;
		*plast++ = '\r';
		break;
	    }

	case '~':
	    plast++;
	    {
		union
		{
		    char *	pc;
		    char	rgc[sizeof(char *)];
		} u1;
		int ic;
		int iHash;
		char *pHash;
		char *pHashPrev;
		char *pString;

		plast[-1] = '\0';
		iHash     = UMIN( MAX_KEY_HASH - 1, plast - 1 - top_string );
		for ( pHash = string_hash[iHash]; pHash; pHash = pHashPrev )
		{
		    for ( ic = 0; ic < (int) sizeof(char *); ic++ )
			u1.rgc[ic] = pHash[ic];
		    pHashPrev = u1.pc;
		    pHash    += sizeof(char *);

		    if ( top_string[sizeof(char *)] == pHash[0]
		    &&   !strcmp( top_string+sizeof(char *)+1, pHash+1 ) )
			return pHash;
		}

		if ( fBootDb )
		{
		    pString		= top_string;
		    top_string		= plast;
		    u1.pc		= string_hash[iHash];
		    for ( ic = 0; ic < (int) sizeof(char *); ic++ )
			pString[ic] = u1.rgc[ic];
		    string_hash[iHash]	= pString;

		    nAllocString += 1;
		    sAllocString += top_string - pString;
		    return pString + sizeof(char *);
		}
		else
		{
		    return str_dup( top_string + sizeof(char *) );
		}
	    }
	}
    }
}



/*
 * Read to end of line (for comments).
 */
void fread_to_eol( FILE *fp )
{
    char c;

    do
    {
	c = getc( fp );
    }
    while ( c != '\n' && c != '\r' );

    do
    {
	c = getc( fp );
    }
    while ( c == '\n' || c == '\r' );

    ungetc( c, fp );
    return;
}



/*
 * Read one word (into static buffer).
 */
char *fread_word( FILE *fp )
{
    static char word[MAX_INPUT_LENGTH];
    char *pword;
    char cEnd;

    do
    {
	cEnd = getc( fp );
    }
    while ( isspace( cEnd ) );

    if ( cEnd == '\'' || cEnd == '"' )
    {
	pword   = word;
    }
    else
    {
	word[0] = cEnd;
	pword   = word+1;
	cEnd    = ' ';
    }

    for ( ; pword < word + MAX_INPUT_LENGTH; pword++ )
    {
	*pword = getc( fp );
	if ( cEnd == ' ' ? isspace((unsigned char) *pword) : *pword == cEnd )
	{
	    if ( cEnd == ' ' )
		ungetc( *pword, fp );
	    *pword = '\0';
	    return word;
	}
    }

    bug( "Fread_word: word too long.", 0 );
    bug( word, 0 );
    exit( 1 );
    return NULL;
}



/*
 * Removes the tildes from a string.
 * Used for player-entered strings that go into disk files.
 */
void smash_tilde( char *str )
{
    for ( ; *str != '\0'; str++ )
    {
	if ( *str == '~' )
	    *str = '-';
    }

    return;
}



/*
 * Compare strings, case insensitive.
 * Return TRUE if different
 *   (compatibility with historical functions).
 */
bool str_cmp( const char *astr, const char *bstr )
{
    if ( astr == NULL )
    {
	bug( "Str_cmp: null astr.", 0 );
	return TRUE;
    }

    if ( bstr == NULL )
    {
	bug( "Str_cmp: null bstr.", 0 );
	return TRUE;
    }

    for ( ; *astr || *bstr; astr++, bstr++ )
    {
	if ( LOWER(*astr) != LOWER(*bstr) )
	    return TRUE;
    }

    return FALSE;
}



/*
 * Compare strings, case insensitive, for prefix matching.
 * Return TRUE if astr not a prefix of bstr
 *   (compatibility with historical functions).
 */
bool str_prefix( const char *astr, const char *bstr )
{
    if ( astr == NULL )
    {
	bug( "Strn_cmp: null astr.", 0 );
	return TRUE;
    }

    if ( bstr == NULL )
    {
	bug( "Strn_cmp: null bstr.", 0 );
	return TRUE;
    }

    for ( ; *astr; astr++, bstr++ )
    {
	if ( LOWER(*astr) != LOWER(*bstr) )
	    return TRUE;
    }

    return FALSE;
}



/*
 * Compare strings, case insensitive, for match anywhere.
 * Returns TRUE is astr not part of bstr.
 *   (compatibility with historical functions).
 */
bool str_infix( const char *astr, const char *bstr )
{
    int sstr1;
    int sstr2;
    int ichar;
    char c0;

    if ( ( c0 = LOWER(astr[0]) ) == '\0' )
	return FALSE;

    sstr1 = strlen(astr);
    sstr2 = strlen(bstr);

    for ( ichar = 0; ichar <= sstr2 - sstr1; ichar++ )
    {
	if ( c0 == LOWER(bstr[ichar]) && !str_prefix( astr, bstr + ichar ) )
	    return FALSE;
    }

    return TRUE;
}



/*
 * Compare strings, case insensitive, for suffix matching.
 * Return TRUE if astr not a suffix of bstr
 *   (compatibility with historical functions).
 */
bool str_suffix( const char *astr, const char *bstr )
{
    int sstr1;
    int sstr2;

    sstr1 = strlen(astr);
    sstr2 = strlen(bstr);
    if ( sstr1 <= sstr2 && !str_cmp( astr, bstr + sstr2 - sstr1 ) )
	return FALSE;
    else
	return TRUE;
}


/*
 * Returns an initial-capped string.
 */
char *capitalize( const char *str )
{
    static char strcap[MAX_STRING_LENGTH];
    int i;

    for ( i = 0; str[i] != '\0'; i++ )
	strcap[i] = LOWER(str[i]);
    strcap[i] = '\0';
    strcap[0] = UPPER(strcap[0]);
    return strcap;
}


/*
 * Append a string to a file.
 */
void append_file( CHAR_DATA *ch, char *file, char *str )
{
    DESCRIPTOR_DATA *d;
    FILE *fp;
    char buf[MAX_STRING_LENGTH];

    if ( IS_NPC(ch) || str[0] == '\0' )
	return;

    sprintf( buf, "[%5d] %s: %s\n",
            ch->in_room ? ch->in_room->vnum : 0, ch->name, str );

    fclose( fpReserve );
    if ( ( fp = fopen( file, "a" ) ) == NULL )
    {
	perror( file );
	send_to_char( "Could not open the file!\n\r", ch );
    }
    else
    {
	fprintf( fp, buf );
	fclose( fp );
    }

    fpReserve = fopen( NULL_FILE, "r" );

    for ( d = descriptor_list; d != NULL; d = d->next )
    {
        if ( d->character == NULL || get_trust( d->character ) < MAX_LEVEL-3
        || d->connected != CON_PLAYING )
            continue;

        send_to_char( buf, d->character );
    }

    return;
}



/*
 * Reports a bug.
 */
void bug( const char *str, int param )
{
    char buf[MAX_STRING_LENGTH];
    FILE *fp;

    if ( fpArea != NULL )
    {
	int iLine;
	int iChar;

	if ( fpArea == stdin )
	{
	    iLine = 0;
	}
	else
	{
	    iChar = ftell( fpArea );
	    fseek( fpArea, 0, 0 );
	    for ( iLine = 0; ftell( fpArea ) < iChar; iLine++ )
	    {
		while ( getc( fpArea ) != '\n' )
		    ;
	    }
	    fseek( fpArea, iChar, 0 );
	}

	sprintf( buf, "[*****] FILE: %s LINE: %d", strArea, iLine );
	log_string( buf );

	if ( ( fp = fopen( "shutdown.txt", "a" ) ) != NULL )
	{
	    fprintf( fp, "[*****] %s\n", buf );
	    fclose( fp );
	}
    }

    strcpy( buf, "[*****] BUG: " );
    sprintf( buf + strlen(buf), str, param );
    log_string( buf );

    fclose( fpReserve );
    if ( ( fp = fopen( BUG_FILE, "a" ) ) != NULL )
    {
	fprintf( fp, "%s\n", buf );
	fclose( fp );
    }
    fpReserve = fopen( NULL_FILE, "r" );

    return;
}



/*
 * Writes a string to the log.
 */
void log_string( const char *str )
{
    DESCRIPTOR_DATA *d;
    char *strtime;

    strtime                    = ctime( &current_time );
    strtime[strlen(strtime)-1] = '\0';
    fprintf( stderr, "%s :: %s\n", strtime, str );

    for ( d = descriptor_list; d != NULL; d = d->next )
    {
        if ( d->character == NULL || get_trust( d->character ) < MAX_LEVEL-3
        || d->connected != CON_PLAYING )
            continue;

        send_to_char( str, d->character );
        send_to_char( "\n\r", d->character );
    }

    return;
}
// I put way too much shit here.
//
// And there's still so much to do.

#if defined(macintosh)
#include <types.h>
#else
#include <sys/types.h>
#endif
#include <stdio.h>
#include <string.h>
#include <time.h>
#include "merc.h"

SKILL_DELAY_DATA *	skill_delay_free;

/* FIXME add adjs to armor/weaps

const struct metal_armor_adj_data metal_armor_adj_table [MAX_METAL_ARMOR_ADJ] =
{
    { "battered", -10, 0 },
    { "bejeweled", 20, 0 },
    { "tarnished", -10, 0 },
    { "silvery", 0, 0 },
    { "black", 0, 0 },
    { "blood-stained", 0, 0 },
    { "burnt", -10, 0 },
    { "carved", 0, 0 },
    { "dirty", 0, 0 },
    { "engraved", 0, 0 },
    { "etched", 0, 0 },
    { "heavy", -20, 40 },
    { "painted", 0, 0 },
    { "polished", 0, 0 },
    { "scarred", 0, 0 },
    { "shiny", 0, 0 },
    { "sturdy", 10, 20 } //17
}


const struct weapon_mod_data weapon_mod_table [MAX_WEAPON_MOD] =
{
    {  "barbed", 0, 0 },
    {  "beautiful", 10, 0 },
    {  "black", 0, 0 },
    {  "blunt", -10, 0 },
    {  "cracked", -10, 0 },
    {  "crude", -20, 0 },
    {  "curved", 0, 0 },
    {  "dull", -10, 0 },
    {  "engraved", 0, 0, 0 },
    {  "etched", 0, 0, 0 },
    {  "fine", 10, 0 },
    {  "heavy", -10, 20 },
    {  "jagged", 0, 0 },
    {  "bejeweled", 20, 0 },
    {  "ornate", 10, 0 },
    {  "polished", 0, 0 },
    {  "ragged", -10, -10, },
    {  "enruned", 20, 20, },
    {  "serrated", 0, 0 },
    {  "shiny", 0, 0, },
    {  "silvery", 0, 0 },
    {  "slender", 10, -20 },
    {  "sturdy", 10, 10 },
    {  "tarnished", -10, 0 },
    {  "warped", -10, 0 },
    {  "well balanced", 10, -10 },
    {  "well crafted", 10, -10 } //27
};


*/


//fixme add flags for metal and wood adjs
const struct box_adj_type box_adj_table [MAX_BOX_ADJ] =
{
{ "acid-pitted",		 			0 },
{ "badly damaged",						0 },
{ "charred", 						0 },
{ "dented", 					0 },
{ "engraved", 					0 },
{ "enruned",		 			0 },
{ "iron-bound",						0 },
{ "plain", 						0 },
{ "blackened", 					0 },
{ "scratched", 					0 },
{ "simple",		 			0 },
{ "sturdy",						0 },
{ "weathered", 						0 }
};

//fixme add value and type wood vs metal
const struct box_matl_type box_matl_table [MAX_BOX_MATL] =
{
{ "brass",		 					0 },
{ "gold",							0 },
{ "crystalline", 						0 },
{ "mithril", 						0 },
{ "glass", 						0 },
{ "alloy",		 					0 },
{ "silver",							0 },
{ "steel", 							0 },
{ "oak", 							0 },
{ "elm", 						0 },
{ "wooden", 						0 },
{ "ironwood",							0 }
};

const struct box_noun_type box_noun_table [MAX_BOX_NOUN] =
{
{ "box", 							0 },
{ "chest", 							0 },
{ "coffer", 						0 },
{ "strongbox", 						0 },
{ "trunk",							0 }
};


const struct container_adj_type container_adj_table [MAX_CONTAINER_ADJ] =
{
{ "ragged",		 			0 },
{ "thick",						0 },
{ "thin", 						0 },
{ "small", 					0 },
{ "big", 					0 },
{ "shiny",		 			0 },
{ "crisp",						0 },
{ "wrinkled", 						0 },
{ "stained", 					0 },
{ "smooth", 					0 },
{ "dark",		 			0 },
{ "patchwork",						0 },
{ "woven", 						0 }
};

/*
canvas
cloth
fur
velvet
brocade
satin
silk
*/

//fixme add value and type leather (cracked) vs cloth (embroidered) vs fur (shaggy) adjs
const struct container_matl_type container_matl_table [MAX_CONTAINER_MATL] =
{
{ "canvas",		 					0 },
{ "cloth",							0 },
{ "fur", 						0 },
{ "velvet", 						0 },
{ "brocade", 						0 },
{ "satin",		 					0 },
{ "silk",							0 },
{ "leather", 							0 },
{ "hide", 							0 },
{ "linen", 						0 },
{ "leather", 						0 },
{ "cloth",							0 }
};

const struct container_noun_type container_noun_table [MAX_CONTAINER_NOUN] =
{
{ "cloak", 							ITEM_EQUIP_CLOAK },
{ "sack", 							ITEM_EQUIP_ONBELT },
{ "pouch", 							ITEM_EQUIP_ONBELT },
{ "satchel", 						ITEM_EQUIP_SHOULDER },
{ "backpack",						ITEM_EQUIP_BACK	 }
};

const struct magic_item_type magic_item_table [MAX_MAGIC_ITEM] =
{
{ "tiny black stone idol",		 			11, 2, 120, 2, 2000 },
{ "small crystal shard",					11, 2, 101, 1, 500 },
{ "large crystal shard", 					11, 2, 107, 1, 1000 },
{ "jagged wand", 							3, 4, 702, 1, 1200 },
{ "cloudy quartz trinket", 					11, 5, 102, 1, 600 },

//healing potions
{ "light red potion",						10, 1, 801, 1, 250 },
{ "pink potion",							10, 1, 803, 1, 500 },
{ "yellow potion",							10, 1, 805, 1, 750 },
{ "pinkish-orange potion",					10, 1, 807, 1, 1500 },
{ "yellowish-green potion",					10, 1, 809, 1, 1750 },
{ "deep red potion",						10, 1, 810, 1, 2000 },
{ "orange potion",							10, 1, 814, 1, 3000 },
{ "green potion",							10, 1, 816, 1, 3500 },

{ "a white scroll",							2, 1, 103, 0, 500 }
};


const struct shield_type shield_table[MAX_SHIELD] = 
{
{"buckler", 45, 160, SHIELD_SMALL, 6, 100 },
{"target shield", 45, 160, SHIELD_SMALL, 6, 100 },
{"shield", 45, 165, SHIELD_MEDIUM, 8, 250 },
{"square shield", 45, 165, SHIELD_MEDIUM, 8, 250 },
{"reinforced shield", 45, 165, SHIELD_MEDIUM, 8, 250 },
{"oval shield", 45, 165, SHIELD_MEDIUM, 8, 250 },
{"round shield", 45, 170, SHIELD_LARGE, 10, 450 },
{"kite shield", 45, 170, SHIELD_LARGE, 10, 450 },
{"tower shield", 45, 175, SHIELD_TOWER, 12, 600 },
{"wall shield", 45, 175, SHIELD_TOWER, 12, 600 }
};

//https://gswiki.play.net/Breakage/saved_posts
//https://gswiki.play.net/WEIGHT_(verb)/saved_posts

//fixme iorake (holy??), star iron (skysteel), etc
const struct metal_type metal_table [MAX_METAL] =
{
{ "bronze", 80, 1, 0, -5, -10 },
{ "iron", 125, 2, 0, 0, 0 },
{ "steel", 100, 3, 0, 10, 20 },
{ "mithril", 90, 100, 5, 20, 40 },
{ "elven", 100, 200, 10, 15, 30 },			//ora
{ "titanium", 70, 300, 12, 20, 40 },		//imflass
{ "dwarven", 150, 330, 15, 24, 65 },		//mein
{ "adamantite", 80, 650, 20, 15, 40 }		//vultite
};


/*
char *name;
	long worn;
	short st;
	short du;
	short v0;
	short v1;
	short v2;
	short v3;
	short weight;
	short cost;
	short RT;
	short AP;
	short CvA1;
	short CvA2;
	short Hindrance;
	short HindranceMax;
*/

const struct armor_type armor_table [MAX_ARMOR] =
{
{ "robes", 				//name
	1|4096, 			//worn
	10, 230,			// st/du
	ARMOR_COVERS_TORSO, 1, 0, 0,			//values 
	8, 50, 			//weight / cost
	0, 0, 25, 15, 0, 0 },
                     // RT, AP, CvA1, CvA2, Hindrance%, Hindrance%Max

{ "light leather", 
	1|4096, 
	10, 250,
	ARMOR_COVERS_TORSO, 1, 1, 1, 
	10, 200,
	0, 0, 25, 15, 0, 0 },
	
{ "full leather", 
	1|4096, 
	20, 285,
	ARMOR_COVERS_TORSO|ARMOR_COVERS_ARMS|ARMOR_COVERS_HANDS, 1, 2, 2, 
	13, 220,
	1, -1, 19, 14, 0, 0 },
	
{ "reinforced leather", 
	1|4096, 
	35, 270, 
	ARMOR_COVERS_TORSO|ARMOR_COVERS_ARMS|ARMOR_COVERS_HANDS|ARMOR_COVERS_LEGS|ARMOR_COVERS_FEET, 1, 3, 3, 
	15, 240,
	2, -5, 18, 13, 2, 4 },
	
{ "double leather", 
	1|4096, 
	35, 290,
	ARMOR_COVERS_EYES|ARMOR_COVERS_HEAD|ARMOR_COVERS_NECK|ARMOR_COVERS_TORSO|ARMOR_COVERS_ARMS|ARMOR_COVERS_HANDS|ARMOR_COVERS_LEGS|ARMOR_COVERS_FEET, 1, 4, 4, 
	16, 280,
	2, -6, 17, 12, 4, 6 },

{ "leather breastplate", 
	1|4096, 
	35, 300,
	ARMOR_COVERS_TORSO, 2, 5, 5, 
	16, 320,
	3, -7, 11, 5, 6, 16 },

{ "cuirboulli leather", 
	1|4096, 
	45, 270,
	ARMOR_COVERS_TORSO|ARMOR_COVERS_ARMS|ARMOR_COVERS_HANDS, 2, 6, 6, 
	17, 360,
	4, -8, 10, 4, 7, 20 },

{ "studded leather", 
	1|4096, 
	20, 285,
	ARMOR_COVERS_TORSO|ARMOR_COVERS_ARMS|ARMOR_COVERS_HANDS|ARMOR_COVERS_LEGS|ARMOR_COVERS_FEET, 2, 7, 7, 
	45, 400,
	5, -10, 9, 3, 6, 24 },

{ "brigandine armor", 
	1|4096, 
	49, 325,
	ARMOR_COVERS_EYES|ARMOR_COVERS_HEAD|ARMOR_COVERS_NECK|ARMOR_COVERS_TORSO|ARMOR_COVERS_ARMS|ARMOR_COVERS_HANDS|ARMOR_COVERS_LEGS|ARMOR_COVERS_FEET, 2, 8, 8,  25, 450,
	-12, 8, 2, 12, 28 },

{ "chain mail", 
	1|4096, 
	55, 350,
	ARMOR_COVERS_TORSO, 3, 9, 9, 
	25, 500, 
	7, -13, 1, -6, 16, 40 },

{ "double chain", 
	1|4096, 
	55, 450,
	ARMOR_COVERS_TORSO|ARMOR_COVERS_ARMS|ARMOR_COVERS_HANDS, 3, 3, 10, 
	25, 550, 
	8, -14, 0, -7, 20, 45 },

{ "augmented chain", 
	1|4096, 
	55, 480, 
	ARMOR_COVERS_TORSO|ARMOR_COVERS_ARMS|ARMOR_COVERS_HANDS|ARMOR_COVERS_LEGS|ARMOR_COVERS_FEET, 3, 3, 11, 
	26, 600, 
	8, -16, -1, -8, 25, 55 },

{ "chain hauberk", 
	1|4096, 
	59, 495,
	ARMOR_COVERS_EYES|ARMOR_COVERS_HEAD|ARMOR_COVERS_NECK|ARMOR_COVERS_TORSO|ARMOR_COVERS_ARMS|ARMOR_COVERS_HANDS|ARMOR_COVERS_LEGS|ARMOR_COVERS_FEET, 3, 3, 12, 
	27, 650, 
	9, -18, -2, -9, 30, 60 },
 
{ "metal breastplate", 
	1|4096, 
	60, 475,
	ARMOR_COVERS_TORSO, 4, 4, 13, 
	23, 700, 
	9, -20, -10, -18, 35, 90 },

{ "augmented breastplate", 
	1|4096, 
	65, 505,
	ARMOR_COVERS_TORSO|ARMOR_COVERS_ARMS|ARMOR_COVERS_HANDS, 4, 4, 14, 
	25, 750, 
	10, -25, -11, -19, 40, 92 },

{ "half plate", 
	1|4096, 
	65, 575,
	ARMOR_COVERS_TORSO|ARMOR_COVERS_ARMS|ARMOR_COVERS_HANDS|ARMOR_COVERS_LEGS|ARMOR_COVERS_FEET, 4, 4, 15, 
	50, 1000, 
	11, -30, -12, -20, 45, 94 },

{ "full plate", 
	1|4096, 
	69, 665,
	ARMOR_COVERS_EYES|ARMOR_COVERS_HEAD|ARMOR_COVERS_NECK|ARMOR_COVERS_TORSO|ARMOR_COVERS_ARMS|ARMOR_COVERS_HANDS|ARMOR_COVERS_LEGS|ARMOR_COVERS_FEET, 4, 4, 16, 
	75, 1500, 
	12, -35, -13, -21, 50, 96 },
	
{ "arm greaves", 
	1|256, 
	25, 185,
	ARMOR_COVERS_ARMS, 1, 1, 17, 
	5, 20, 
	0, 0, 0, 0, 0, 0 },
	
{ "leg greaves", 
	1|65536, 
	25, 200,
	ARMOR_COVERS_LEGS, 1, 1, 18, 
	5, 20, 
	0, 0, 0, 0, 0, 0 },
	
{ "gauntlets", 
	1|512, 
	25, 150,
	ARMOR_COVERS_HANDS, 1, 1, 19, 
	2, 10, 
	0, 0, 0, 0, 0, 0 },
	
{ "boots", 
	1|262144, 
	25, 165,
	ARMOR_COVERS_FEET, 1, 1, 20, 
	2, 10, 
	0, 0, 0, 0, 0, 0 },
	
{ "helm", 
	1|8, 
	30, 200,
	ARMOR_COVERS_HEAD|ARMOR_COVERS_EYES, 1, 1, 21, 
	2, 10, 
	0, 0, 0, 0, 0, 0 },
	
{ "aventail", 
	1|32, 
	45, 300,
	ARMOR_COVERS_NECK, 1, 1, 22, 
	2, 10, 
	0, 0, 0, 0, 0, 0 },
	
{ "greathelm", 
	1|8|32, 
	40, 290,
	ARMOR_COVERS_HEAD|ARMOR_COVERS_EYES|ARMOR_COVERS_NECK, 1, 1, 23, 
	4, 20, 
	0, 0, 0, 0, 0, 0 },
};

const struct gem_type gem_table [MAX_GEM] =
{
{ "clear sapphire", 0, 5, 20 },
{ "clear topaz", 0, 5, 20 },
{ "blue topaz", 0, 5, 20 },
{ "brown zircon", 0, 5, 20 },
{ "quartz crystal", 0, 5, 20 },
{ "rock crystal", 0, 5, 20 },
{ "clear zircon", 0, 5, 20 },
{ "star diopside", 0, 5, 20 },	// 7

{ "carnelian quartz", 1, 30, 60 },
{ "iridescent labradorite stone", 1, 30, 60 },
{ "polished black coral", 1, 30, 60  },
{ "almandine garnet", 1, 30, 60  }, // 11


{ "green aventurine stone", 2, 40, 70 },
{ "black jasper", 2, 40, 70 },
{ "cat's eye quartz", 2, 40, 70 },
{ "yellow zircon", 2, 40, 70 },
{ "polished blue coral", 2, 40, 70 }, // 16


{ "blue cordierite", 3, 50, 85 },
{ "banded sardonyx stone", 3, 50, 85 },
{ "piece of amber", 3, 50, 85 },
{ "yellow jasper", 3, 50, 85 },
{ "pink tourmaline", 3, 50, 85 },	// 21

{ "fire opal", 4, 70, 100 },
{ "dark red-green bloodstone", 4, 70, 100 },
{ "red jasper", 4, 70, 100 },
{ "blue coral", 4, 70, 100 },		// 25


{ "clear tourmaline", 5, 90, 200 },
{ "rose quartz", 5, 90, 200},
{ "black tourmaline", 5, 90, 200 },
{ "golden beryl gem", 5, 90, 200 },
{ "citrine quartz", 5, 90, 200 },
{ "amethyst", 5, 90, 200 },			// 31

{ "light pink morganite stone", 6, 120, 340 },
{ "red spinel", 6, 120, 340 },
{ "bright chrysoberyl gem", 6, 120, 340 },
{ "green tourmaline", 6, 120, 340 },
{ "green zircon", 6, 120, 340 },
{ "blue spinel", 6, 120, 340 },
{ "blue tourmaline", 6, 120, 340 },		// 38


{ "green garnet", 7, 250, 350 },		// 39


{ "pink rhodocrosite stone", 8, 450, 650 },
{ "turquoise stone", 8, 450, 650 },			// 41


{ "green malachite stone", 9, 550, 800 },
{ "blue lapis lazuli", 9, 550, 800 },
{ "golden topaz", 9, 550, 800 },				// 44


{ "polished red coral", 10, 625, 875 },
{ "pink topaz", 10, 625, 875 },
{ "polished pink coral", 10, 625, 875 },		// 47


{ "smoky topaz", 11, 800, 1000 },
{ "aquamarine gem", 11, 800, 1000 }			// 49

};

//is it just me or is this easier to read than an area file?
const struct creature_type creature_table [MAX_CREATURE] =
{
{ "giant rat",				1,		5,	  -10, 		3, 3, 25, ACT_ANIMAL|ACT_BITE, 0, SPEED_NORMAL, CLASS_FIGHTER, "rat pelt", 666 },  
{ "gazzpat",				1,		10,		0, 		3, 3, 35, ACT_ANIMAL|ACT_BITE, 0, SPEED_NORMAL, CLASS_FIGHTER, "gazzpat hide", 666 },
{ "kobold",					1,		10,		10,		3, 3, 30, ACT_SCAVENGER, 0, SPEED_NORMAL, CLASS_FIGHTER, "kobold skin", 666 },
{ "lesser ghoul",			1,		10,		5, 		3, 3, 25, ACT_SCAVENGER|ACT_CLAW|ACT_SEXLESS, AFF_UNDEAD, SPEED_NORMAL, CLASS_FIGHTER, NULL, 666 },
{ "skeleton",				1,		10,		0, 		3, 3, 30, ACT_SCAVENGER|ACT_CLAW, AFF_UNDEAD, SPEED_NORMAL, CLASS_FIGHTER, "skeleton bone", 666 },
{ "runtling",				1,		10,		5, 		3, 3, 45, ACT_SCAVENGER, 0, SPEED_NORMAL, CLASS_FIGHTER, NULL, 666 },
{ "giant centipede",		1,		10,		10, 	3, 3, 40, ACT_ANIMAL|ACT_BITE, 0, SPEED_NORMAL, CLASS_FIGHTER, "centipede carapace", 666 },
{ "cobra",					2,		30,		40, 	6, 6, 20, ACT_ANIMAL|ACT_BITE, 0, SPEED_NORMAL, CLASS_FIGHTER, "cobra skin", 666 },
{ "ghost",					2,		30,		24, 	6, 6, 50, ACT_SCAVENGER, AFF_UNDEAD|AFF_NONCORP, SPEED_NORMAL, CLASS_FIGHTER, NULL, 666 },
{ "shadowkin",				2,		20,		30, 	6, 6, 60, ACT_SCAVENGER|ACT_SEXLESS, 0, SPEED_NORMAL, CLASS_FIGHTER, NULL, 666 },
{ "goblin",					2,		40,		30, 	6, 6, 50, ACT_SCAVENGER, 0, SPEED_NORMAL, CLASS_FIGHTER, NULL, 666 },
{ "phantom",				2,		30,		20, 	6, 6, 45, ACT_SCAVENGER|ACT_SPELLCASTER|ACT_SEXLESS, AFF_UNDEAD|AFF_NONCORP, SPEED_NORMAL, CLASS_FIGHTER, NULL, 666 },
{ "greater gazzpat",		3,		35,		25, 	3, 3, 30, ACT_ANIMAL|ACT_CHARGE, 0, SPEED_NORMAL, CLASS_FIGHTER, "gazzpat tail", 666 },
{ "hobgoblin",				3,		50,		25, 	9, 9, 60, ACT_SCAVENGER, 0, SPEED_NORMAL, CLASS_FIGHTER, "hobgobln scalp", 666 },
{ "gnoll",	        		3,		60,		35, 	9, 9, 60, ACT_SCAVENGER, 0, SPEED_NORMAL, CLASS_FIGHTER, NULL, 666 },
{ "greater ghoul",			3,		40,		20, 	9, 9, 60, ACT_SCAVENGER|ACT_CLAW|ACT_SEXLESS, AFF_UNDEAD, SPEED_NORMAL, CLASS_FIGHTER, "zombie nail", 666 },
{ "ice skeleton",			3,		40,		35, 	9, 9, 50, ACT_SCAVENGER, AFF_UNDEAD, SPEED_NORMAL, CLASS_FIGHTER, "icy skeleton bone", 666 },
{ "troglodyte",				3,		45,		30, 	9, 9, 65, ACT_SCAVENGER, 0, SPEED_NORMAL, CLASS_FIGHTER, NULL, 666 },
{ "tunnel worm",			3,		60,		45, 	9, 9, 145, ACT_ANIMAL|ACT_BITE, 0, SPEED_NORMAL, CLASS_FIGHTER, "white worm skin", 666 },
{ "will-o-the-wisp",		3,		45,		40, 	9, 9, 70, ACT_SCAVENGER|ACT_CHARGE|ACT_SPELLCASTER|ACT_SEXLESS, 0, SPEED_NORMAL, CLASS_MAGE, NULL, 666 },
{ "horned black rabbit",	4,		55,		55, 	12, 12, 80, ACT_ANIMAL|ACT_SCAVENGER|ACT_BITE, 0, SPEED_NORMAL, CLASS_FIGHTER, "large black bunny foot", 666 },
{ "giant centipede",		4,		60,		45, 	12, 12, 80, ACT_ANIMAL|ACT_SCAVENGER|ACT_BITE|ACT_SEXLESS, 0, SPEED_NORMAL, CLASS_FIGHTER, "think centipede carapace", 666 },
{ "revenant",				4,		60,		45, 	12, 12, 115, ACT_SCAVENGER, AFF_UNDEAD|AFF_NONCORP, SPEED_NORMAL, CLASS_FIGHTER, NULL, 666 },
{ "lesser orc",				4, 		60,		30, 	12, 12, 60, ACT_SCAVENGER, 0, SPEED_NORMAL, CLASS_FIGHTER, "green orc hide", 666 },
{ "whip-tailed shrieker",	4, 		65,		45, 	12, 12, 60, ACT_ANIMAL|ACT_CHARGE, 0, SPEED_NORMAL, CLASS_FIGHTER, "shrieker tail" , 666 },
{ "gnoll ranger",			5, 		60,		40, 	15, 15, 60, ACT_SCAVENGER, 0, SPEED_NORMAL, CLASS_FIGHTER, NULL, 666 },
{ "wolverine",				5, 		50,		50, 	15, 15, 50, ACT_ANIMAL|ACT_BITE, 0, SPEED_NORMAL, CLASS_FIGHTER, "wolverine pelt", 666 },
{ "greater orc",			5, 		80,   	50, 	15, 15, 80, ACT_SCAVENGER, 0, SPEED_NORMAL, CLASS_FIGHTER, "red orc hide", 666 },
{ "darkling",				5, 		50,  	30, 	15, 15, 90, ACT_SCAVENGER|ACT_SEXLESS, 0, SPEED_NORMAL, CLASS_FIGHTER, NULL, 666 },
{ "mummy",					6, 	    40,   	30, 	18, 18, 100, ACT_SCAVENGER, AFF_UNDEAD, SPEED_NORMAL, CLASS_FIGHTER, "mummy shroud", 666 },
{ "fire ghost",				6, 		60,  	20, 	18, 18, 85, ACT_SCAVENGER|ACT_SPELLCASTER|ACT_SEXLESS, AFF_UNDEAD|AFF_NONCORP, SPEED_NORMAL, CLASS_MAGE, NULL, 666 },
{ "water witch",			6, 		50,  	30, 	18, 18, 100, ACT_SCAVENGER|ACT_SPELLCASTER, AFF_UNDEAD|AFF_NONCORP, SPEED_NORMAL, CLASS_MAGE, NULL, 666 },
{ "cockatrice",				6, 		50,   	45, 	18, 18, 70, ACT_CLAW, 0, SPEED_NORMAL, CLASS_FIGHTER, "cockatrice tail", 666 },
{ "yellow troll",			6, 		60,  	25, 	18, 18, 180, ACT_SCAVENGER|ACT_CLAW, 0, SPEED_NORMAL, CLASS_FIGHTER, "yellow troll hide", 666 },
{ "blood-beast",			7, 		65,  	70, 	21, 21, 120, ACT_SCAVENGER|ACT_CLAW|ACT_TERMINATOR, 0, SPEED_NORMAL, CLASS_FIGHTER, "blood-beast skin", 666 },
{ "basilisk",				7, 		70,   	70, 	21, 21, 130, ACT_BITE, 0, SPEED_NORMAL, CLASS_FIGHTER, "basilisk skin", 666 },
{ "gnoll thief",			7, 		80,   	30, 	21, 21, 80, ACT_SCAVENGER, 0, SPEED_NORMAL, CLASS_FIGHTER, NULL, 666 },
{ "lizardman",				7, 		75,   	55, 	21, 21, 150, ACT_SCAVENGER, 0, SPEED_NORMAL, CLASS_FIGHTER, "lizardman hide", 666 },
{ "sea witch",				7, 		65,   	70, 	21, 21, 125, ACT_SCAVENGER|ACT_SPELLCASTER, 0, SPEED_NORMAL, CLASS_MAGE, NULL, 666 },
{ "giant spider",			8, 		75,  	40, 	24, 24, 100, ACT_ANIMAL|ACT_CHARGE, 0, SPEED_NORMAL, CLASS_MAGE, "spider leg", 666 },
{ "brown bear",				8, 		70,   	50, 	24, 24, 270, ACT_ANIMAL|ACT_CLAW, 0, SPEED_NORMAL, CLASS_FIGHTER, "bear hide", 666 },
{ "fanged gibberling",		8, 		70,   	55, 	24, 24, 130, ACT_ANIMAL|ACT_CHARGE, 0, SPEED_NORMAL, CLASS_FIGHTER, NULL, 666 },
{ "rust golem",				8, 		100,   	30, 	24, 24, 110, ACT_AUTOMATON|ACT_POUND|ACT_SEXLESS, 0, SPEED_NORMAL, CLASS_FIGHTER, NULL, 666 },
{ "rotting zombie",			9, 		60,   	75, 	27, 27, 125, ACT_CLAW, AFF_UNDEAD, SPEED_NORMAL, CLASS_FIGHTER, "zombie hide", 666 },
{ "manticore",				9, 		75,   	55, 	27, 27, 150, ACT_CLAW, 0, SPEED_NORMAL, CLASS_FIGHTER, "manticore tail", 666 },
{ "moldering zombie bear",	9, 		95,   	68, 	27, 27, 230, ACT_CLAW, AFF_UNDEAD, SPEED_NORMAL, CLASS_FIGHTER, NULL, 666 },
{ "black troll",			10,   	120,    70, 	30, 30, 250, ACT_SCAVENGER|ACT_CLAW, 0, SPEED_NORMAL, CLASS_FIGHTER, "black troll hide", 666 },
{ "mountain lion",			10,    	85,  	100, 	30, 30, 150, ACT_ANIMAL|ACT_BITE, 0, SPEED_NORMAL, CLASS_FIGHTER, "mountain lion pelt", 666 },
{ "hunch-backed ogre",		10,    	80,   	27, 	30, 30, 150, ACT_POUND, 0, SPEED_NORMAL, CLASS_FIGHTER, NULL, 666 },
{ "grey orc",				10,    	90,   	90, 	30, 30, 130, ACT_SCAVENGER|ACT_SPELLCASTER, 0, SPEED_NORMAL, CLASS_FIGHTER, "grey orc hide", 666 },
{ "lizardman chieftain",	11,    	95,  	120, 	33, 33, 175, ACT_SCAVENGER, 0, SPEED_NORMAL, CLASS_FIGHTER, "lizardman hide", 666 },
{ "ghost wolf",    			11,   	100,   	65, 	33, 33, 200, ACT_BITE|ACT_TERMINATOR, AFF_UNDEAD|AFF_NONCORP, SPEED_NORMAL, CLASS_FIGHTER, NULL, 666 },
{ "green troll",    		11,   	110,   	70, 	33, 33, 205, ACT_SCAVENGER|ACT_CLAW, 0, SPEED_NORMAL, CLASS_FIGHTER, "green orc hide", 666 },  
{ "crystal golem",    		12,    	90,   	70, 	36, 36, 220, ACT_POUND|ACT_SEXLESS, 0, SPEED_NORMAL, CLASS_FIGHTER, NULL, 666 },
{ "dark orc",    			12,    	85,  	122, 	36, 36, 170, ACT_SCAVENGER, 0, SPEED_NORMAL, CLASS_FIGHTER, "dark orc hide", 666 },
{ "red troll",    			12,   	150,   	62, 	36, 36, 265, ACT_SCAVENGER|ACT_CLAW, 0, SPEED_NORMAL, CLASS_FIGHTER, "red troll hide", 666 },
{ "zombie giant",     		12,   	110,   	40, 	36, 36, 290, ACT_SCAVENGER, AFF_UNDEAD, SPEED_NORMAL, CLASS_FIGHTER, "thick zombie hide", 666 },
{ "lesser tree fiend",    	13,   	100,   	70, 	39, 39, 210, ACT_POUND|ACT_SPELLCASTER|ACT_SEXLESS, AFF_UNDEAD, SPEED_NORMAL, CLASS_MAGE, NULL, 666 },
{ "titan",    				13,   	140,   	85, 	39, 39, 320, ACT_SCAVENGER|ACT_POUND, 0, SPEED_NORMAL, CLASS_FIGHTER, NULL, 666 },
{ "ghostly specter",    	14,    	90,   	170, 	42, 42, 250, ACT_SCAVENGER|ACT_SEXLESS, AFF_UNDEAD|AFF_NONCORP|ACT_SPELLCASTER, SPEED_NORMAL, CLASS_MAGE, NULL, 666 },
{ "ogre",     				14,   	140,  	105, 	42, 42, 200, ACT_SCAVENGER|ACT_POUND, 0, SPEED_NORMAL, CLASS_FIGHTER, NULL, 666 }, 
{ "ice golem",    			15,   	100,  	100, 	45, 45, 175, ACT_POUND|ACT_SEXLESS, 0, SPEED_NORMAL, CLASS_FIGHTER, NULL, 666 },
{ "shadowy wraith",    		15,   	110,  	110, 	45, 45, 240, ACT_SCAVENGER|ACT_SPELLCASTER|ACT_SEXLESS, AFF_UNDEAD|AFF_NONCORP, SPEED_NORMAL, CLASS_MAGE, NULL, 666 },
{ "undead hulk",    		15,   	165,  	135, 	45, 45, 160, ACT_AUTOMATON|ACT_SCAVENGER|ACT_SEXLESS, AFF_UNDEAD, SPEED_NORMAL, CLASS_FIGHTER, NULL, 666 },
{ "fire wight",    			16,   	100,  	100, 	48, 48, 195, ACT_POUND|ACT_SPELLCASTER|ACT_SEXLESS, AFF_NONCORP, SPEED_NORMAL, CLASS_MAGE, NULL, 666 },
{ "ghoul king",    			16,   	125,   	80, 	48, 48, 180, ACT_SCAVENGER|ACT_SPELLCASTER, AFF_UNDEAD, SPEED_NORMAL, CLASS_FIGHTER, "ghoul scalp", 666 },
{ "shadow stalker",    		17,   	165,   	90, 	51, 51, 210, ACT_SCAVENGER|ACT_TERMINATOR, 0, SPEED_NORMAL, CLASS_FIGHTER, NULL, 666 },  
{ "shadow assassin",    	18,   	150,   	90, 	54, 54, 180, ACT_SCAVENGER|ACT_TERMINATOR, 0, SPEED_NORMAL, CLASS_FIGHTER, NULL, 666 },
{ "storm giant",    		18,   	140,  	105, 	54, 54, 300, ACT_SCAVENGER|ACT_POUND|ACT_SPELLCASTER, 0, SPEED_NORMAL, CLASS_MAGE, "speckled grey giant hide", 666 },
{ "frost giant",    		19,   	145,  	100, 	57, 57, 275, ACT_SCAVENGER|ACT_POUND|ACT_SPELLCASTER, 0, SPEED_NORMAL, CLASS_MAGE, "snow white giant hide", 666 },
{ "arachnid horror",    	19,   	120,   	90, 	57, 57, 425, ACT_SCAVENGER|ACT_SEXLESS, 0, SPEED_NORMAL, CLASS_FIGHTER, "purplish egg sac", 666 },
{ "steel soldier",  		20,   	160,  	105, 	60, 60, 300, ACT_AUTOMATON|ACT_POUND|ACT_SEXLESS, 0, SPEED_NORMAL, CLASS_FIGHTER, NULL, 666 },
{ "wight lord",    			20,   	240,  	160, 	60, 60, 400, ACT_SCAVENGER, AFF_UNDEAD, SPEED_NORMAL, CLASS_FIGHTER, NULL, 666 },
{ "greater tree fiend",    	21,   	140,  	240, 	63, 63, 450, ACT_CLAW|ACT_SPELLCASTER|ACT_SEXLESS, AFF_UNDEAD|AFF_NONCORP, SPEED_NORMAL, CLASS_MAGE, NULL, 666 },
{ "Chaos apprentice",    	21,   	207,  	202, 	63, 63, 425, ACT_SCAVENGER|ACT_SPELLCASTER, 0, SPEED_NORMAL, CLASS_FIGHTER, NULL, 666 },
{ "red satyr",    			23,  	190,  	210, 	69, 69, 300, ACT_SCAVENGER, 0, SPEED_NORMAL, CLASS_FIGHTER, "red satyr hide", 666 },
{ "Chaos servant",     		23,   	219,  	238, 	69, 69, 500, ACT_SCAVENGER|ACT_SPELLCASTER, 0, SPEED_NORMAL, CLASS_PRIEST, NULL, 666 },
{ "hell hound",    			24,   	230,  	192, 	72, 72, 500, ACT_ANIMAL|ACT_BITE|ACT_TERMINATOR, 0, SPEED_NORMAL, CLASS_FIGHTER, "bright red hide", 666 },
{ "Chaos liturgist",    	25,   	221,  	224, 	75, 75, 525, ACT_SCAVENGER|ACT_SPELLCASTER, 0, SPEED_NORMAL, CLASS_PRIEST, NULL, 666 },
{ "minor demon",    		25,   	250,  	220, 	75, 75, 550, ACT_POUND|ACT_SPELLCASTER, 0, SPEED_NORMAL, CLASS_FIGHTER, "demon hide", 666 },
{ "giant spider",   		27,   	249,  	284, 	81, 81, 575, ACT_ANIMAL|ACT_CLAW|ACT_SEXLESS, 0, SPEED_NORMAL, CLASS_FIGHTER, "large egg sac", 666 },
{ "centaur mystic",    		31,   	260,  	235, 	100, 100, 700, ACT_SCAVENGER|ACT_SPELLCASTER, 0, SPEED_NORMAL, CLASS_MAGE, "centaur hide", 666 },
{ "centaur crusader",    	33,   	275,  	240, 	120, 120, 800, ACT_SCAVENGER, 0, SPEED_NORMAL, CLASS_MAGE, "centaur hide", 666 },
{ "centaur battlemage",    	35,   	285,  	245, 	140, 140, 900, ACT_SCAVENGER|ACT_CHARGE|ACT_SEXLESS, 0, SPEED_NORMAL, CLASS_MAGE, "centaur hide", 666 },
{ "centaur paladin",    	37,   	290,  	250, 	120, 120, 800, ACT_SCAVENGER|ACT_CHARGE|ACT_SEXLESS, 0, SPEED_NORMAL, CLASS_PRIEST, "centaur hide", 666 },
{ "centaur warchief",    	39,   	295,  	255, 	140, 140, 900, ACT_SCAVENGER|ACT_CHARGE|ACT_SEXLESS, 0, SPEED_NORMAL, CLASS_FIGHTER, "centaur hide", 666 },
{ "mein golem",    			38,   	300,  	300, 	100, 100, 700, ACT_AUTOMATON|ACT_SCAVENGER|ACT_POUND|ACT_SEXLESS, 0, SPEED_NORMAL, CLASS_FIGHTER, NULL, 666 },
{ "stone gargoyle",    		39,   	350,  	325, 	120, 120, 800, ACT_AUTOMATON|ACT_SCAVENGER|ACT_POUND|ACT_SEXLESS, 0, SPEED_NORMAL, CLASS_FIGHTER, NULL, 666 },
{ "mithril minion",    		43,   	375,  	350, 	140, 140, 900, ACT_AUTOMATON|ACT_SCAVENGER|ACT_POUND|ACT_SEXLESS, 0, SPEED_NORMAL, CLASS_FIGHTER, NULL, 666 },
{ "mithril sentinel",  		46,   	390,  	365, 	160, 160, 4175, ACT_AUTOMATON|ACT_SCAVENGER|ACT_POUND|ACT_SEXLESS, 0, SPEED_NORMAL, CLASS_PRIEST, NULL,666 },
{ "mithril guardian",  		49,   	450,  	400, 	200, 220, 4175, ACT_AUTOMATON|ACT_SCAVENGER|ACT_SPELLCASTER|ACT_SEXLESS, 0, SPEED_NORMAL, CLASS_MAGE, NULL, 666 }
};

//fixme alt names
const struct weapon_type weapon_table [MAX_WEAPON] =
{
    {
	"closed fist", -1, -1, WEAPON_TYPE_BRAWLING, WEAPON_SPEED_VERY_FAST, 100, 75, 40, 36, 32, 0, 0
    }, //0
    {
	"dagger", 18, 195, WEAPON_TYPE_EDGED, WEAPON_SPEED_VERY_FAST, 250, 200, 100, 125, 75, 1, 6  //1
    },
	{
	"short sword", 70, 185, WEAPON_TYPE_EDGED, WEAPON_SPEED_FAST,  350, 240, 15, 125, 75, 3, 70
    },
	{
	"rapier", 30, 100, WEAPON_TYPE_EDGED, WEAPON_SPEED_FAST,  325, 225, 125, 125, 75, 3, 60
    },
	{
	"scimitar", 60, 150, WEAPON_TYPE_EDGED, WEAPON_SPEED_FAST,  375, 260, 210, 200, 165, 4, 70
    },
	{
	"longsword", 65, 160, WEAPON_TYPE_EDGED, WEAPON_SPEED_NORMAL,  425, 275, 225, 200, 175, 5, 100
    },
	{
	"broadsword", 75, 160, WEAPON_TYPE_EDGED, WEAPON_SPEED_NORMAL,  450, 300, 250, 225, 200, 5, 110
    },
	
	{
	"falchion", 75, 160, WEAPON_TYPE_EDGED, WEAPON_SPEED_NORMAL,  450, 325, 250, 250, 175, 5, 125
    },
	
	{
	"hand axe", 70, 160, WEAPON_TYPE_EDGED, WEAPON_SPEED_NORMAL,  420, 300, 275, 240, 210, 4, 100
    },
	
	{
	"whip", 10, 75, WEAPON_TYPE_BLUNT, WEAPON_SPEED_VERY_FAST, 275, 150, 90, 100, 35, 2, 20
	},
	
	{
	"cudgel", 8, 130, WEAPON_TYPE_BLUNT, WEAPON_SPEED_FAST, 350, 275, 200, 225, 150, 4, 10
	}, //10
	
	{
	"mace", 65, 250, WEAPON_TYPE_BLUNT, WEAPON_SPEED_NORMAL, 400, 300, 225, 250, 175, 5, 100
	},
	
	{
	"ball and chain", 75, 175, WEAPON_TYPE_BLUNT, WEAPON_SPEED_NORMAL, 400, 300, 225, 260, 180, 5, 110
	},
	
	{
	"war hammer", 60, 155, WEAPON_TYPE_BLUNT, WEAPON_SPEED_NORMAL, 410, 290, 250, 275, 200, 5, 110
	},
	
	{
	"morning star", 65, 250, WEAPON_TYPE_BLUNT, WEAPON_SPEED_NORMAL, 425, 325, 275, 300, 225, 5, 125
	},


	{
	"staff", -1, -1, WEAPON_TYPE_TWOHANDED, WEAPON_SPEED_NORMAL, 510, 420, 390, 250, 200, 3, 50
    },
	
    {
	"greatsword", 18, 195, WEAPON_TYPE_TWOHANDED, WEAPON_SPEED_VERY_SLOW, 625, 500, 500, 350, 275, 6, 160
    },
	
	{
	"flail", 70, 185, WEAPON_TYPE_TWOHANDED, WEAPON_SPEED_SLOW,  575, 425, 400, 350, 275, 6, 170
    },
	
	{
	"greataxe", 30, 100, WEAPON_TYPE_TWOHANDED, WEAPON_SPEED_VERY_SLOW, 650, 475, 500, 375, 275, 6, 160
    },
	
	{
	"maul", 60, 150, WEAPON_TYPE_TWOHANDED, WEAPON_SPEED_SLOW,  550, 425, 425, 375, 300, 8, 150 //19
    },
	
	
	//polearms
	
	//ranged? thrown?
	
	{
	"bite", -1, -1, WEAPON_TYPE_NATURAL, WEAPON_SPEED_NORMAL, 400, 375, 375, 325, 300, 0, 0
	}, // 15
	
	{
	"charge", -1, -1, WEAPON_TYPE_NATURAL, WEAPON_SPEED_NORMAL, 175, 175, 150, 175, 150, 0, 0
	},
	
	{
	"claw", -1, -1, WEAPON_TYPE_NATURAL, WEAPON_SPEED_NORMAL, 225, 200, 200, 175, 175, 0, 0
	},
	
	{
	"pound", -1, -1, WEAPON_TYPE_NATURAL, WEAPON_SPEED_SLOW, 425, 350, 325, 325, 275, 0, 0
	}
	
};	


const struct bolt_type bolt_table [MAX_BOLT_SPELL] =
{
    {
	"surge of electricity", DAMAGE_TYPE_SHOCK, 150, 133, 111, 122, 128
	},
	{
	"stream of water", 	DAMAGE_TYPE_WATER, 455, 345, 283, 242, 173
	},
	{
	"stream of acid", 	DAMAGE_TYPE_ACID, 525, 383, 314, 350, 197
	},
	{
	"stream of fire", 	DAMAGE_TYPE_FIRE, 667, 455, 345, 323, 303
	},
	{
	"ball of ice",		DAMAGE_TYPE_COLD, 445, 350, 245, 217, 208
	},
	{
	"ball of fire", 	DAMAGE_TYPE_FIRE, 400, 333, 270, 256, 244
	},
	{
	"bolt of lightning", DAMAGE_TYPE_SHOCK, 750, 555, 433, 415, 433
	},
	{
	"massive boulder", DAMAGE_TYPE_CRUSH, 710, 520, 460, 435, 440
	},
	{
	"disintegrating ray", DAMAGE_TYPE_DISINTEGRATE, 525, 383, 314, 350, 197
	},
	{
	"hissing cloud of steam", DAMAGE_TYPE_FIRE, 600, 550, 385, 333, 267
	}
};

// based on slash chest table...missing ancillary wounds to back and abdomen
// fixme currently unused
const struct crit_type crit_table [MAX_BODY][MAX_CRIT] =
{
		{
	{
		"$n totally misses $N.", 0, STATUS_NONE, WOUND_NONE, BLEED_NONE
	},

	{
		"$n grazes $N.", 1, STATUS_NONE, WOUND_RANK_1, BLEED_NONE
	},

	{
		"$n barely hits $N.", 10, STATUS_NONE, WOUND_RANK_1, BLEED_NONE
	},

	{
		"$n hits $N.", 15, STATUS_STUN_1, WOUND_RANK_1, BLEED_NONE
	},

	{
		"$n hits $N hard.", 20, STATUS_STUN_3, WOUND_RANK_2, BLEED_NONE
	},

	{
		"$n hits $N very hard.", 25, STATUS_STUN_5, WOUND_RANK_2, BLEED_2PER
	},
	
	{
		"$n hits $N extremely hard.", 45, STATUS_STUN_6, WOUND_RANK_3, BLEED_4PER
	},
	
	{
		"$n massacres $N!", 60, STATUS_STUN_8, WOUND_RANK_3, BLEED_6PER
	},
	
	{
		"$n annihilates $N!", 65, STATUS_DEATH, WOUND_RANK_3, BLEED_8PER	
	},
	
	{
		"$n brutalizes $N!", 70, STATUS_DEATH, WOUND_RANK_3, BLEED_10PER
	}
	},
	
		{
	{
		"Quick slash at $N's right eye. Strike lands but misses target.", 0, STATUS_NONE, WOUND_NONE, BLEED_NONE
	},

	{
		"Slashing strike near forehead nicks an eyebrow! That must sting!", 3, STATUS_NONE, WOUND_RANK_1, BLEED_NONE
	},

	{
		"Gash to $N's right eyebrow. That's going to be quite a shiner!", 3, STATUS_STUN_1, WOUND_RANK_1, BLEED_NONE
	},

	{
		"Grazing slash to $N's face! Scratch to its eyelids. \"When blood gets in your eyes...\"", 5, STATUS_STUN_3, WOUND_RANK_2, BLEED_NONE
	},

	{
		"Upward slash gouges $N's cheek! Right eye lost! Pity.", 20, STATUS_STUN_5, WOUND_RANK_3, BLEED_NONE
	},

	{
		"Slash strikes $N's right eye. Seems there was a brain there after all.", 25, STATUS_DEATH, WOUND_RANK_3, BLEED_NONE
	},
	
	{
		"Slash to head destroys $N's right eye! Doesn't do its brain any good either. 	", 30, STATUS_DEATH, WOUND_RANK_3, BLEED_NONE
	},
	
	{
		"Slash to $N's right eye! Vitreous fluid spews forth! Seeya!", 40, STATUS_DEATH, WOUND_RANK_3, BLEED_NONE
	},
	
	{
		"Horrifying slash to $N's head! Right eye sliced open! Brain pureed!", 45, STATUS_DEATH, WOUND_RANK_3, BLEED_NONE
	},
	
	{
		"Blast to $N's head destroys right eye! Brain obliterated! Disgusting, but painful only for a second.", 50, STATUS_DEATH, WOUND_RANK_3, BLEED_NONE
	}
	},
	
			{
	{
		"Quick slash to $N's left eye. Strike lands but misses target.", 0, STATUS_NONE, WOUND_NONE, BLEED_NONE
	},

	{
		"Slashing strike near forehead nicks an eyelid! That must sting!", 3, STATUS_NONE, WOUND_RANK_1, BLEED_NONE
	},

	{
		"Gash to $N's left eyebrow. That's going to be quite a shiner!", 3, STATUS_STUN_1, WOUND_RANK_1, BLEED_NONE
	},

	{
		"Grazing slash to $N's face! Scratches its left eye. Ouch!", 5, STATUS_STUN_3, WOUND_RANK_2, BLEED_NONE
	},

	{
		"Upward slash gouges $N's cheek! Left eye lost! Pity.", 20, STATUS_STUN_5, WOUND_RANK_3, BLEED_NONE
	},

	{
		"Slash strikes $N's left eye. Seems there was a brain there after all.", 25, STATUS_DEATH, WOUND_RANK_3, BLEED_NONE
	},
	
	{
		"Slash to head destroys $N's left eye! Doesn't do its brain any good either.", 30, STATUS_DEATH, WOUND_RANK_3, BLEED_NONE
	},
	
	{
		"Slash to $N's left eye! Vitreous fluid spews forth! Seeya!", 40, STATUS_DEATH, WOUND_RANK_3, BLEED_NONE
	},
	
	{
		"Horrifying slash to $N's head! Left eye sliced open! Brain pureed!", 45, STATUS_DEATH, WOUND_RANK_3, BLEED_NONE
	},
	
	{
		"Blast to $N's head destroys left eye! Brain obliterated! Disgusting, but painful only for a second.", 50, STATUS_DEATH, WOUND_RANK_3, BLEED_NONE
	}
	},
	
		{
	{
		"Flashy swing! Too bad it only bopped $N's nose.", 0, STATUS_NONE, WOUND_NONE, BLEED_NONE
	},

	{
		"Quick slash catches $N's cheek!  Dimples are always nice.", 5, STATUS_NONE, WOUND_RANK_1, BLEED_NONE
	},

	{
		"Blade slashes across $N's face!  Nice nose job.", 10, STATUS_STUN_1, WOUND_RANK_1, BLEED_NONE
	},

	{
		"Blow to head!", 15, STATUS_STUN_3, WOUND_RANK_2, BLEED_2PER
	},

	{
		"Quick flick of the wrist!  $N is slashed across $S forehead! ", 20, STATUS_STUN_6, WOUND_RANK_2, BLEED_2PER
	},

	{
		"Hard blow to $N's ear!  Deep gash and a terrible headache! ", 25, STATUS_STUN_8, WOUND_RANK_3, BLEED_4PER
	},
	
	{
		"Gruesome slash opens $N's forehead!  Grey matter spills forth! ", 30, STATUS_DEATH, WOUND_RANK_3, BLEED_4PER
	},
	
	{
		"Wild upward slash removes $N's face from $S skull!  Interesting way to die.", 35, STATUS_DEATH, WOUND_RANK_3, BLEED_6PER	
	},
	
	{
		"Horrible slash to $N's head! Brain matter goes flying!  Looks like $E never felt a thing.", 40, STATUS_DEATH, WOUND_RANK_3, BLEED_6PER	
	},
	
	{
		"Gruesome, slashing blow to the side of $N's head!  Skull split open!  Brain (and life) vanishes in a fine mist.", 50, STATUS_DEATH, WOUND_RANK_3, BLEED_8PER
	}
	},
	
		{
	{
		"Close shave!  $N takes a quick step back.", 0, STATUS_NONE, WOUND_NONE, BLEED_NONE
	},

	{
		"Attack hits $N's throat but doesn't break the skin.  Close!", 2, STATUS_NONE, WOUND_RANK_1, BLEED_NONE
	},

	{
		"Strike dents $N's larynx.  Swallowing will be fun.", 5, STATUS_NONE, WOUND_RANK_1, BLEED_NONE
	},

	{
		"Deft swing strikes $N's neck.  Maybe not fatal but it's sure distracting.", 10, STATUS_STUN_1, WOUND_RANK_1, BLEED_NONE
	},

	{
		"Strong slash to throat nicks a few blood vessels.", 12, STATUS_STUN_3, WOUND_RANK_2, BLEED_4PER
	},

	{
		"Fast slash to $N's neck exposes $S windpipe.  Quick anatomy lesson, anyone?", 15, STATUS_STUN_5, WOUND_RANK_2, BLEED_4PER
	},
	
	{
		"Deep slash to $N's neck severs an artery!  $N chokes to death on $S own blood.", 20, STATUS_STUN_6, WOUND_RANK_3, BLEED_6PER
	},
	
	{
		"Gruesome slash to $N's throat!  That stings... for about a second.", 25, STATUS_STUN_8, WOUND_RANK_3, BLEED_6PER
	},
	
	{
		"Awful slash nearly decapitates $N!  That's one way to lose your head.", 30, STATUS_DEATH, WOUND_RANK_3, BLEED_8PER
	},
	
	{
		"Incredible slash to $N's neck!  Throat and vocal cords destroyed!  Zero chance of survival.", 40, STATUS_DEATH, WOUND_RANK_3, BLEED_10PER
	}
	},
	
		{
	{
		"Weak slash across chest!  Slightly less painful than heartburn.", 0, STATUS_NONE, WOUND_NONE, BLEED_NONE
	},

	{
		"Deft slash across chest draws blood!  $N takes a deep breath.", 1, STATUS_NONE, WOUND_RANK_1, BLEED_NONE
	},

	{
		"Slash to $N's chest!  That heart's not broken, it's only scratched.", 10, STATUS_NONE, WOUND_RANK_1, BLEED_NONE
	},

	{
		"Slash to $N's chest.  Breathe deep, it'll feel better in a minute.", 15, STATUS_STUN_1, WOUND_RANK_1, BLEED_NONE
	},

	{
		"Slashing blow to chest knocks $N back a few paces!", 20, STATUS_STUN_3, WOUND_RANK_2, BLEED_4PER
	},

	{
		"Crossing slash to the chest catches $N's attention!", 25, STATUS_STUN_5, WOUND_RANK_2, BLEED_4PER
	},
	
	{
		"Hard slash to $N's side opens its spleen!", 45, STATUS_STUN_6, WOUND_RANK_3, BLEED_4PER
	},
	
	{
		"Quick, powerful slash!  $N's chest is ripped open!", 60, STATUS_STUN_8, WOUND_RANK_3, BLEED_4PER	
	},
	
	{
		"Slash to $N''s ribs opens a sucking chest wound!", 65, STATUS_DEATH, WOUND_RANK_3, BLEED_6PER	
	},
	
	{
		"Wicked slash slices open $N's chest!  Heart and lung pureed!  Sickening!", 70, STATUS_DEATH, WOUND_RANK_3, BLEED_6PER
	}
	},
	
		{
	{
		"Glancing blow to $N's back. That could have been better.", 0, STATUS_NONE, WOUND_NONE, BLEED_NONE
	},

	{
		"Weak slash to $N's lower back!", 3, STATUS_NONE, WOUND_RANK_1, BLEED_NONE
	},

	{
		"Feint to the left goes astray as $N dodges! You scratch my back...", 10, STATUS_NONE, WOUND_RANK_1, BLEED_NONE
	},

	{
		"Slash along $N's lower back.", 15, STATUS_NONE, WOUND_RANK_1, BLEED_NONE
	},

	{
		"Slash to $N's lower back! Pain shoots up along $N's spine.", 20, STATUS_STUN_2, WOUND_RANK_1, BLEED_1PER
	},

	{
		"Feint left spins $N around! Jagged slash to lower back.", 25, STATUS_STUN_3, WOUND_RANK_2, BLEED_1PER
	},
	
	{
		"$N twists away but is caught with a hard slash! Back is broken!", 30, STATUS_STUN_5|STATUS_KNOCKDOWN, WOUND_RANK_3, BLEED_2PER
	},
	
	{
		"Deft slash! $N is spun around and hit hard in its lower back.", 50, STATUS_STUN_8|STATUS_KNOCKDOWN, WOUND_RANK_3, BLEED_2PER	
	},
	
	{
		"Slash to $N's lower back! Kidneys sliced and diced! Death is slow and painful.", 60, STATUS_DEATH, WOUND_RANK_3, BLEED_4PER	
	},
	
	{
		"Masterful slash to $N's lower back! Spinal cord and life are just memories now.", 75, STATUS_DEATH, WOUND_RANK_3, BLEED_4PER
	}
	},
	
		{
	{
		"Weak slash to $N's right arm. That doesn't even sting.", 0, STATUS_NONE, WOUND_NONE, BLEED_NONE
	},

	{
		"Quick slash to $N's upper right arm! Just a nick.", 3, STATUS_NONE, WOUND_RANK_1, BLEED_NONE
	},

	{
		"Hesitant slash to $N's upper right arm! Just a scratch.", 7, STATUS_NONE, WOUND_RANK_1, BLEED_NONE
	},

	{
		"Slash to $N's right arm! Slices neatly through the skin and meets bone!", 8, STATUS_STUN_1, WOUND_RANK_1, BLEED_NONE
	},

	{
		"Powerful slash just cracks $N's weapon arm!", 10, STATUS_STUN_2, WOUND_RANK_2, BLEED_2PER
	},

	{
		"Deep slash to $N's right forearm!", 15, STATUS_STUN_3, WOUND_RANK_2, BLEED_2PER
	},
	
	{
		"Quick, hard slash to $N's right arm! \"CRACK\"", 20, STATUS_STUN_5, WOUND_RANK_2, BLEED_4PER
	},
	
	{
		"Hard slash to $N's side! Right arm no longer available for use.", 25, STATUS_STUN_6, WOUND_RANK_3, BLEED_4PER	
	},
	
	{
		"Spectacular slash! $N's right arm is neatly amputated!", 35, STATUS_STUN_8, WOUND_RANK_3, BLEED_4PER
	},
	
	{
		"Awesome slash sever $N's right arm! A jagged stump is all that remains!", 40, STATUS_STUN_10, WOUND_RANK_3, BLEED_6PER
	}
	},
	
		{
	{
		"Near-miss! That'll hurt tomorrow.", 0, STATUS_NONE, WOUND_NONE, BLEED_NONE
	},

	{
		"Diagonal slash to $N's weapon arm. Strike misses but bruises a few knuckles.", 1, STATUS_NONE, WOUND_RANK_1, BLEED_NONE
	},

	{
		"Wild slash bounces off the back of $N's hand.", 3, STATUS_NONE, WOUND_RANK_1, BLEED_NONE
	},

	{
		"Feint to $N's head! Quick flick at its weapon hand! Nasty cut to right hand!", 5, STATUS_NONE, WOUND_RANK_1, BLEED_NONE
	},

	{
		"Strong slash to $N's right hand cuts deep.", 7, STATUS_STUN_1, WOUND_RANK_2, BLEED_1PER
	},

	{
		"Slash to $N's weapon hand! Several fingers fly!", 5, STATUS_STUN_2, WOUND_RANK_2, BLEED_1PER
	},
	
	{
		"Rapped $N's knuckles hard! Right hand sounds broken.", 10, STATUS_STUN_3, WOUND_RANK_2, BLEED_1PER
	},
	
	{
		"Jagged slash to $N's right arm! Cut clean through at the wrist. Need a hand?", 15, STATUS_STUN_4, WOUND_RANK_3, BLEED_2PER
	},
	
	{
		"Powerful slash trims $N's fingernails... and the remainder of its right hand!", 25, STATUS_STUN_5, WOUND_RANK_3, BLEED_2PER	
	},
	
	{
		"Off-balanced slash! Enough force to sever $N's right hand! Amazing!", 30, STATUS_STUN_10, WOUND_RANK_3, BLEED_4PER
	}
	},
	
		{
	{
		"Hard blow, but deflected. Not much damage.", 0, STATUS_NONE, WOUND_NONE, BLEED_NONE
	},

	{
		"Quick slash to $N's upper left arm! Just a nick.", 3, STATUS_NONE, WOUND_RANK_1, BLEED_NONE
	},

	{
		"Slash to $N's shield arm! Shears off a thin layer of skin!", 7, STATUS_NONE, WOUND_RANK_1, BLEED_NONE
	},

	{
		"Glancing slash to $N's shield arm!", 8, STATUS_STUN_1, WOUND_RANK_1, BLEED_NONE
	},

	{
		"Powerful slash just cracks $N's shield arm!", 10, STATUS_STUN_2, WOUND_RANK_2, BLEED_2PER
	},

	{
		"Deep slash to $N's left forearm!", 15, STATUS_STUN_3, WOUND_RANK_2, BLEED_2PER
	},
	
	{
		"Off-balance slash to $N's left arm shatters its elbow. \"CRUNCH\"", 20, STATUS_STUN_5, WOUND_RANK_2, BLEED_4PER
	},
	
	{
		"Hard slash to $N's side! Left arm no longer available for use.", 25, STATUS_STUN_6, WOUND_RANK_3, BLEED_4PER	
	},
	
	{
		"Spectacular slash! $N's left arm is neatly amputated!", 35, STATUS_STUN_8, WOUND_RANK_3, BLEED_4PER
	},
	
	{
		"Awesome slash severs $N's left arm! A jagged stump is all that remains!", 40, STATUS_STUN_10, WOUND_RANK_3, BLEED_6PER
	}
	},
	
		{
	{
		"Near-miss!  Knuckles kissed but little damage.", 0, STATUS_NONE, WOUND_NONE, BLEED_NONE
	},

	{
		"Slash to $N's shield arm.  Strike trims off a few fingernails.", 1, STATUS_NONE, WOUND_RANK_1, BLEED_NONE
	},

	{
		"Wild slash scratches the back of $N's hand.", 3, STATUS_NONE, WOUND_RANK_1, BLEED_NONE
	},

	{
		"Slice to $N's left fingers.  Nice move.", 5, STATUS_NONE, WOUND_RANK_1, BLEED_NONE
	},

	{
		"Deep cut to $N's left hand!  Seems to have broken some fingers too.", 7, STATUS_STUN_1, WOUND_RANK_2, BLEED_1PER
	},

	{
		"Slash to $N's shield hand!  Several fingers fly!", 5, STATUS_STUN_2, WOUND_RANK_2, BLEED_1PER
	},
	
	{
		"Rapped the knuckles hard!  Left hand sounds broken.", 10, STATUS_STUN_3, WOUND_RANK_2, BLEED_1PER
	},
	
	{
		"Jagged slash to the $N's left arm!  Cut clean through at the wrist.  Need a hand?", 15, STATUS_STUN_4, WOUND_RANK_3, BLEED_2PER	
	},
	
	{
		"Powerful slash trims $N's fingernails... and the remainder of its left hand!", 25, STATUS_STUN_5, WOUND_RANK_3, BLEED_2PER	
	},
	
	{
		"Off-balanced slash!  Enough force to sever $N's left hand!  Amazing!", 30, STATUS_STUN_10, WOUND_RANK_3, BLEED_4PER
	}
	},
	
		{
	{
		"Light slash to $N's abdomen!  Barely nicked.", 0, STATUS_NONE, WOUND_NONE, BLEED_NONE
	},

	{
		"Awkward slash to $N's stomach!  Everyone needs another belly button.", 5, STATUS_NONE, WOUND_RANK_1, BLEED_NONE
	},

	{
		"Smooth slash to $N's hip!  Nice crunching sound.", 10, STATUS_NONE, WOUND_RANK_1, BLEED_NONE
	},

	{
		"Hard slash to belly severs a few nerve endings.", 15, STATUS_STUN_3, WOUND_RANK_2, BLEED_2PER
	},

	{
		"Diagonal slash leaves a bloody trail across the $N's torso.", 20, STATUS_STUN_3, WOUND_RANK_2, BLEED_4PER
	},

	{
		"$N is backed up by a strong slash to $S abdomen!", 25, STATUS_STUN_5, WOUND_RANK_2, BLEED_4PER
	},
	
	{
		"Deep slash to $N's right side! Several inches of padding sliced off hip.... From the inside!", 30, STATUS_STUN_6, WOUND_RANK_3, BLEED_6PER
	},
	
	{
		"Amazing slash to $N's belly!  Nothing quite like that empty feeling inside.", 50, STATUS_STUN_8, WOUND_RANK_3, BLEED_6PER	
	},
	
	{
		"Bloody slash to $N's side!  Instant death, due to lack of intestines.", 60, STATUS_DEATH, WOUND_RANK_3, BLEED_8PER	
	},
	
	{
		"Terrible slash to $N's side!  Entrails spill out, onto the ground!  Death can be SO messy.", 75, STATUS_DEATH, WOUND_RANK_3, BLEED_8PER
	}
	},
	
		{
	{
		"Quick feint to $N's right leg! Little extra damage.", 0, STATUS_NONE, WOUND_NONE, BLEED_NONE
	},

	{
		"Slash to $N's right leg hits high! Kinda makes your knees weak, huh?", 5, STATUS_NONE, WOUND_RANK_1, BLEED_NONE
	},

	{
		"Banged $N's right shin. That'll raise a good welt.", 10, STATUS_NONE, WOUND_RANK_1, BLEED_NONE
	},

	{
		"Downward slash across $N's right thigh! Might not scar.", 10, STATUS_STUN_1, WOUND_RANK_1, BLEED_NONE
	},

	{
		"Deep, bloody slash to $N's right thigh!", 17, STATUS_STUN_3|STATUS_KNOCKDOWN, WOUND_RANK_2, BLEED_2PER
	},

	{
		"Quick, powerful slash to $N's right knee!", 20, STATUS_STUN_5|STATUS_KNOCKDOWN, WOUND_RANK_2, BLEED_2PER
	},
	
	{
		"Strong slash to $N's right leg!  Muscles exposed!  Not a pretty sight.", 25, STATUS_STUN_6|STATUS_KNOCKDOWN, WOUND_RANK_2, BLEED_3PER
	},
	
	{
		"Wild downward slash severs $N's right leg!  Bloody stump, anyone?", 30, STATUS_STUN_8|STATUS_KNOCKDOWN, WOUND_RANK_3, BLEED_4PER
	},
	
	{
		"Powerful slash!  $N's right leg is severed at the knee!", 40, STATUS_STUN_9|STATUS_KNOCKDOWN, WOUND_RANK_3, BLEED_4PER
	},
	
	{
		"Powerful slash leaves $N without a right leg!", 45, STATUS_STUN_10|STATUS_KNOCKDOWN, WOUND_RANK_3, BLEED_6PER
	}
	},
	
		{
	{
		"Quick feint to $N's right foot! Little extra damage.", 0, STATUS_NONE, WOUND_NONE, BLEED_NONE
	},

	{
		"Slash to $N's right foot hits low! Heel lightly bruised.", 5, STATUS_NONE, WOUND_RANK_1, BLEED_NONE
	},

	{
		"Banged $N's right heel. Uncomfortable.", 10, STATUS_NONE, WOUND_RANK_1, BLEED_NONE
	},

	{
		"Downward slash across $N's right ankle! Might not scar.", 10, STATUS_STUN_1, WOUND_RANK_1, BLEED_NONE
	},

	{
		"Deep, bloody slash to $N's right ankle!", 17, STATUS_STUN_3|STATUS_KNOCKDOWN, WOUND_RANK_2, BLEED_2PER
	},

	{
		"Quick, powerful slash to $N's right ankle!", 20, STATUS_STUN_5|STATUS_KNOCKDOWN, WOUND_RANK_2, BLEED_2PER
	},
	
	{
		"Strong slash to $N's right foot!  A bloody bruise results.", 25, STATUS_STUN_6|STATUS_KNOCKDOWN, WOUND_RANK_2, BLEED_3PER
	},
	
	{
		"Wild downward slash severs $N's right foot!  Hop to it.", 30, STATUS_STUN_8|STATUS_KNOCKDOWN, WOUND_RANK_3, BLEED_4PER
	},
	
	{
		"Powerful slash!  $N's right foot is severed at the knee!", 40, STATUS_STUN_9|STATUS_KNOCKDOWN, WOUND_RANK_3, BLEED_4PER
	},
	
	{
		"Powerful slash leaves $N without a right foot!", 45, STATUS_STUN_10|STATUS_KNOCKDOWN, WOUND_RANK_3, BLEED_6PER
	}
	},
	
			{
	{
		"Light, bruising slash to $N's left thigh.", 0, STATUS_NONE, WOUND_NONE, BLEED_NONE
	},

	{
		"Slash to $N's left leg hits high!  Kinda makes your knees weak, huh?", 5, STATUS_NONE, WOUND_RANK_1, BLEED_NONE
	},

	{
		"Banged $N's left shin.  That'll raise a good welt.", 10, STATUS_NONE, WOUND_RANK_1, BLEED_NONE
	},

	{
		"Downward slash across $N's left thigh!  Gouges bone!", 10, STATUS_STUN_1, WOUND_RANK_1, BLEED_NONE
	},

		{
		"Deft slash to the $N's left leg digs deep!  Bone is chipped!", 17, STATUS_STUN_3|STATUS_KNOCKDOWN, WOUND_RANK_2, BLEED_2PER
	},

	{
		"Quick, powerful slash to $N's left knee!", 20, STATUS_STUN_5|STATUS_KNOCKDOWN, WOUND_RANK_2, BLEED_2PER
	},
	
	{
		"Weak diagonal slash catches $N's left knee!  It is dislocated.", 25, STATUS_STUN_6|STATUS_KNOCKDOWN, WOUND_RANK_2, BLEED_3PER
	},
	
	{
		"Wild downward slash severs $N's left leg!  Bloody stump, anyone?", 30, STATUS_STUN_8|STATUS_KNOCKDOWN, WOUND_RANK_3, BLEED_4PER
	},
	
	{
		"Powerful slash!  $N's left leg is severed at the knee!", 40, STATUS_STUN_9|STATUS_KNOCKDOWN, WOUND_RANK_3, BLEED_4PER
	},
	
	{
		"Powerful slash leaves $N without a left leg!", 45, STATUS_STUN_10|STATUS_KNOCKDOWN, WOUND_RANK_3, BLEED_6PER
	}

	},
	
			{
	{
		"Quick feint to $N's left foot! Little extra damage.", 0, STATUS_NONE, WOUND_NONE, BLEED_NONE
	},

	{
		"Slash to $N's left foot hits low! Heel lightly bruised.", 5, STATUS_NONE, WOUND_RANK_1, BLEED_NONE
	},

	{
		"Banged $N's left heel. Uncomfortable.", 10, STATUS_NONE, WOUND_RANK_1, BLEED_NONE
	},

	{
		"Downward slash across $N's left ankle! Might not scar.", 10, STATUS_STUN_1, WOUND_RANK_1, BLEED_NONE
	},

	{
		"Deep, bloody slash to $N's left ankle!", 17, STATUS_STUN_3|STATUS_KNOCKDOWN, WOUND_RANK_2, BLEED_2PER
	},

	{
		"Quick, powerful slash to $N's left ankle!", 20, STATUS_STUN_5|STATUS_KNOCKDOWN, WOUND_RANK_2, BLEED_2PER
	},
	
	{
		"Strong slash to $N's left foot!  A bloody bruise results.", 25, STATUS_STUN_6|STATUS_KNOCKDOWN, WOUND_RANK_2, BLEED_3PER
	},
	
	{
		"Wild downward slash severs $N's left foot!  Hop to it.", 30, STATUS_STUN_8|STATUS_KNOCKDOWN, WOUND_RANK_3, BLEED_4PER
	},
	
	{
		"Powerful slash!  $N's left foot is severed at the knee!", 40, STATUS_STUN_9|STATUS_KNOCKDOWN, WOUND_RANK_3, BLEED_4PER
	},
	
	{
		"Powerful slash leaves $N without a left foot!", 45, STATUS_STUN_10|STATUS_KNOCKDOWN, WOUND_RANK_3, BLEED_6PER
	}
	},
	
	
		{
	{
		"$n rank 0 central nervous system $N", 0, STATUS_NONE, WOUND_NONE, BLEED_NONE
	},

	{
		"$n rank 1 central nervous system $N", 0, STATUS_NONE, WOUND_NONE, BLEED_NONE
	},

	{
		"$n rank 2 central nervous system $N", 0, STATUS_NONE, WOUND_NONE, BLEED_NONE
	},

	{
		"$n rank 3 central nervous system $N", 0, STATUS_NONE, WOUND_NONE, BLEED_NONE
	},

	{
		"$n rank 4 central nervous system $N", 0, STATUS_NONE, WOUND_NONE, BLEED_NONE
	},

	{
		"$n rank 5 central nervous system $N", 0, STATUS_NONE, WOUND_NONE, BLEED_NONE
	},
	
	{
		"$n rank 6 central nervous system $N", 0, STATUS_NONE, WOUND_NONE, BLEED_NONE
	},
	
	{
		"$n rank 7 central nervous system $N", 0, STATUS_NONE, WOUND_NONE, BLEED_NONE
	},
	
	{
		"$n rank 8 central nervous system $N", 0, STATUS_NONE, WOUND_NONE, BLEED_NONE
	},
	
	{
		"$n rank 9 central nervous system $N", 0, STATUS_NONE, WOUND_NONE, BLEED_NONE
	}
	}
};

char *	const	body_name	[] =
{
	"none",
    "right eye",
	"left eye",
	"head",
	"neck",
	"chest",
	"back",
	"right arm",
	"right hand",
	"left arm",
	"left hand",
	"abdomen",
	"right leg",
	"right foot",
	"left leg",
	"left foot",
	"central nervous system"
};

/*
#define STAT_STRENGTH		1
#define STAT_ENDURANCE		2
#define STAT_DEXTERITY		3
#define STAT_SPEED			4
#define STAT_WILLPOWER		5
#define STAT_POTENCY		6
#define STAT_JUDGEMENT		7
#define STAT_INTELLIGENCE	8
#define STAT_WISDOM			9
#define STAT_CHARM			10


#define RACE_HUMAN 0
#define RACE_HIGH_MAN 1
#define RACE_HALF_ELF 2
#define RACE_HIGH_ELF 3
#define RACE_WOOD_ELF 4
#define RACE_DARK_ELF 5
#define RACE_DWARF 6
#define RACE_HALFLING 7
#define RACE_GNOME 7

*/

//race bonuses always adds up to +15
int stat_bonus ( int stat, CHAR_DATA *ch, int which )
{
	short what = (stat-50)/2;
	
	if( !IS_NPC(ch) )
	{
	switch( ch->race )
	{
		case RACE_HUMAN: 
		if( which == STAT_STRENGTH || which == STAT_INTELLIGENCE || which == STAT_JUDGEMENT )
				return what+5;
		else
				return what;
		
		case RACE_HIGH_MAN: 
		if( which == STAT_STRENGTH )
				return what+15;
		else if( which == STAT_ENDURANCE )
				return what+10;
		else if( which == STAT_CHARM )
				return what+5;
		else if( which == STAT_DEXTERITY || which == STAT_SPEED || which == STAT_POTENCY || which == STAT_JUDGEMENT	)
				return what-5;
		else
				return what;
	
		case RACE_HIGH_ELF: 
		if( which == STAT_SPEED )
				return what+15;
		else if( which == STAT_CHARM )
				return what+10;
		else if( which == STAT_DEXTERITY || which == STAT_POTENCY )
				return what+5;
		else if( which == STAT_WILLPOWER	)
				return what-15;
		else
				return what;
		
		case RACE_HALF_ELF:
		if( which == STAT_SPEED)
				return what+10;
		else if( which == STAT_DEXTERITY || which == STAT_CHARM )
				return what+5;
		else if( which == STAT_CHARM )
				return what+5;
		else if( which == STAT_WILLPOWER )
				return what-5;
		else
			return what;
		
		case RACE_WOOD_ELF:
		if( which == STAT_DEXTERITY)
				return what+10;
		else if( which == STAT_SPEED || which == STAT_POTENCY )
				return what+5;
		else if( which == STAT_WILLPOWER	)
				return what-5;
		else
			return what;
		
		case RACE_DARK_ELF: 
		if( which == STAT_DEXTERITY || which == STAT_POTENCY )
				return what+10;
		else if( which == STAT_SPEED || which == STAT_INTELLIGENCE || which == STAT_WISDOM )
				return what+5;
		else if( which == STAT_ENDURANCE || which == STAT_CHARM )
				return what-5;
		else if( which == STAT_WILLPOWER )
				return what-10;
		else
				return what;
		
		case RACE_DWARF: 
		if( which == STAT_ENDURANCE )
				return what+15;
		else if( which == STAT_STRENGTH || which == STAT_WILLPOWER )
				return what+10;
		else if( which == STAT_CHARM || which == STAT_POTENCY )
				return what-10;
		else if( which == STAT_SPEED	)
				return what-5;
		else
				return what;
			
		case RACE_HALFLING: 
		if( which == STAT_STRENGTH )
				return what-15;
		else if( which == STAT_ENDURANCE || which == STAT_SPEED || which == STAT_INTELLIGENCE )
				return what+10;
		else if( which == STAT_CHARM )
				return what-5;
		else if( which == STAT_DEXTERITY)
				return what+15;
		else if( which == STAT_WILLPOWER)
				return what+5;
		else
				return what;	
			
		case RACE_GNOME:  //forest, not burghal, don't need 2 -15 str races
		if( which == STAT_STRENGTH)
				return what-10;
		else if( which == STAT_ENDURANCE || which == STAT_SPEED )
				return what+10;
		else if( which == STAT_DEXTERITY || which == STAT_WILLPOWER || which ==STAT_JUDGEMENT || which == STAT_WISDOM  )
				return what+5;
		else if( which == STAT_CHARM	)
				return what-5;
		else
				return what;
			
	default: 
	return what;
	}
	}
		
	return what;
}

int skill_bonus( int ranks )
{
	if( ranks > 40 )
		return 140 + (ranks-40);
	else if( ranks > 30 )
		return 120 + (ranks-30)*2;
	else if( ranks > 20 )
		return 90 + (ranks-20)*3;
	else if( ranks > 10 )
		return 50 + (ranks-10)*4;
	else if( ranks > 0 )
		return ranks*5;
	else
		return 0;
}

int random_hit_location( )
{
	switch( number_range( 1, 100 ) )
	{
		case 1: case 2: 
		return BODY_R_EYE;
		
		case 3: case 4: 
		return BODY_L_EYE;
		
		case 5: case 6: 
		case 7: case 8: 
		case 9: case 10: 
		case 11:
		return BODY_HEAD;
		
		case 12: case 13: 
		case 14: case 15: 
		case 16: case 17:
		return BODY_NECK;
		
		case 18: case 19: 
		case 20: case 21: 
		case 22: case 23:
		case 24: case 25: 
		case 26: case 27: 
		case 28: case 29: 
		case 30: case 77:
		return BODY_CHEST;
		
		case 31: case 32: 
		case 33: case 34: 
		case 35: case 36:
		case 37: case 38: 
		case 39: case 40: 
		case 100: 
		return BODY_BACK;
		
		case 41: case 42: 
		case 43: case 44: 
		case 45: case 46:
		case 47: case 48: 
		case 49:
		return BODY_R_ARM;
		
		case 50: case 51: 
		case 52: case 53: 
		return BODY_R_HAND;
		
		case 54: case 55: 
		case 56: case 57: 
		case 58: case 59:
		case 60: case 61: 
		case 62:
		return BODY_L_ARM;
		
		case 63: case 64: 
		case 65: case 66: 
		return BODY_L_HAND;
		
		case 67: case 68: 
		case 69: case 70: 
		case 71: case 72:
		case 73: case 74: 
		case 75: case 76:
		return BODY_ABDOMEN;
		
		case 89: case 90: 
		case 91: case 92: 
		case 93: case 94:
		case 95: case 96: 
		return BODY_R_LEG;
		
		case 97: case 98: 
		case 99: 
		return BODY_R_FOOT;
		
		case 78: case 79: 
		case 80: case 81: 
		case 82: case 83:
		case 84: case 85:
		return BODY_L_LEG;
		
		case 86: case 87: 
		case 88:
		return BODY_L_FOOT;
		
		default:
		return BODY_NONE;
	}
}


int get_armor_flag( int hit_loc )
{
switch (hit_loc)
{
	case BODY_R_EYE: case BODY_L_EYE: 
	return ARMOR_COVERS_EYES;
	
	case BODY_HEAD: 
	return ARMOR_COVERS_HEAD;
	
	case BODY_NECK:
	return ARMOR_COVERS_NECK;
	
	case BODY_CHEST: case BODY_BACK: case BODY_ABDOMEN: 
	return ARMOR_COVERS_TORSO;
	
	case BODY_R_ARM: case BODY_L_ARM: 
	return ARMOR_COVERS_ARMS;
	
	case BODY_R_HAND: case BODY_L_HAND: 
	return ARMOR_COVERS_HANDS;
	
	case BODY_R_LEG: case BODY_L_LEG: 
	return ARMOR_COVERS_LEGS;
	
	case BODY_R_FOOT: case BODY_L_FOOT: 
	return ARMOR_COVERS_FEET;
	
	case BODY_NONE: case BODY_CNS: case MAX_BODY: default: 
	return ARMOR_COVERS_TORSO;
}
}

short get_armor_hit_loc( CHAR_DATA *ch, int hit_loc )
{
	OBJ_DATA *obj;
	short highest = 0;
	int armor_flag = get_armor_flag( hit_loc );

    for ( obj = ch->carrying; obj != NULL; obj = obj->next_content )
    {
		if( obj->wear_loc != EQUIP_NONE
		&& obj->wear_loc != EQUIP_RIGHTHAND
		&& obj->wear_loc != EQUIP_LEFTHAND
		&& obj->item_type == ITEM_ARMOR && 
			IS_SET( obj->value[0], armor_flag ) && obj->value[1] > highest )
				highest = obj->value[1];			
    }
	
	if( highest > ARMOR_GROUP_PLATE )
		highest = ARMOR_GROUP_PLATE;
		
	if( highest < ARMOR_GROUP_SKIN )
		highest = ARMOR_GROUP_SKIN;

//if wearing leather, greaves are leather.
	obj = get_eq_char( ch, EQUIP_CHEST );
		
	if( obj != NULL && highest > obj->value[1] )
		highest = obj->value[1];			
	
	return highest;
}

OBJ_DATA *get_armor_hit_obj( CHAR_DATA *ch, int hit_loc )
{
	OBJ_DATA *obj;
	int armor_flag = get_armor_flag( hit_loc );

    for ( obj = ch->carrying; obj != NULL; obj = obj->next_content )
    {
		if( obj->wear_loc != EQUIP_NONE
		&& obj->wear_loc != EQUIP_RIGHTHAND
		&& obj->wear_loc != EQUIP_LEFTHAND
		&& obj->item_type == ITEM_ARMOR && 
			IS_SET( obj->value[0], armor_flag ) )
			{
				return obj;			
			}
    }
	
	return NULL;
}

short get_damage_factor( short weapon_type, short armor_group )
{
	weapon_type -= TYPE_HIT;

	if( weapon_type < 0 || weapon_type > MAX_WEAPON-1 )
		return 1000;
		
	switch( armor_group )
	{
		case ARMOR_GROUP_PLATE: 
		return weapon_table[weapon_type].damage_factor_plate;
		case ARMOR_GROUP_CHAIN: 
		return weapon_table[weapon_type].damage_factor_chain;
		case ARMOR_GROUP_SCALE: 
		return weapon_table[weapon_type].damage_factor_scale;
		case ARMOR_GROUP_LEATHER: 
		return weapon_table[weapon_type].damage_factor_leather;
		case ARMOR_GROUP_SKIN: 
		default:
		return weapon_table[weapon_type].damage_factor_skin;
	}
}

short get_bolt_damage_factor( short bolt_type, short armor_group )
{
	bolt_type -= TYPE_BOLT_SPELL;

	if( bolt_type < 0 || bolt_type > MAX_BOLT_SPELL-1 )
		return 1000;
		
	switch( armor_group )
	{
		case ARMOR_GROUP_PLATE: 
		return bolt_table[bolt_type].damage_factor_plate;
		case ARMOR_GROUP_CHAIN: 
		return bolt_table[bolt_type].damage_factor_chain;
		case ARMOR_GROUP_SCALE: 
		return bolt_table[bolt_type].damage_factor_scale;
		case ARMOR_GROUP_LEATHER: 
		return bolt_table[bolt_type].damage_factor_leather;
		case ARMOR_GROUP_SKIN: 
		default:
		return bolt_table[bolt_type].damage_factor_skin;
	}
}

char *dt_to_name( short dt )
{
	if( dt >= TYPE_BOLT_SPELL )
	{
	dt -= 2000;
	
	if( dt > -1 && dt < MAX_BOLT_SPELL )
	return bolt_table[dt].name;
	else
	return "bolt";
	}
	else if( dt >= TYPE_HIT )
	{
	dt -= 1000;
	
	if( dt > -1 && dt < MAX_WEAPON )
	return weapon_table[dt].name;
	else
	return "weapon";
	}
	else
	return "attack";
}

char *ag_to_name( short ag)
{
	switch( ag )
	{
		case ARMOR_GROUP_PLATE: return "plate";
		case ARMOR_GROUP_CHAIN: return "chain";
		case ARMOR_GROUP_SCALE: return "scale";
		case ARMOR_GROUP_LEATHER: return "leather";
		case ARMOR_GROUP_SKIN: default: return "skin";
	}
}

short get_divisor( short ag )
{
	switch( ag )
	{
		case ARMOR_GROUP_PLATE: return 11;
		case ARMOR_GROUP_CHAIN: return 9;
		case ARMOR_GROUP_SCALE: return 7;
		case ARMOR_GROUP_LEATHER: return 6;
		case ARMOR_GROUP_SKIN: default: return 5;
	}
}


void do_body( CHAR_DATA *ch, char *argument )
{
	int iBody = 1;
	
	while( iBody < MAX_BODY )
	{
	printf_to_char( ch, "%s:      injury: %d   armor: %d\r\n", body_name[iBody], ch->body[iBody],
		get_armor_hit_loc( ch, iBody ) );
	iBody++;
	}
	
	return;
}

//fixme racial mods vs spell spheres
int get_td( CHAR_DATA *ch )
{
	return (ch->level*3) + ch->td;
}

int get_cs( CHAR_DATA *ch )
{
	return (ch->level*4.5) + 25 + ch->cs;
}


int get_bolt_as( CHAR_DATA *ch )
{
	return ch->as+35;
}

int get_bolt_ds( CHAR_DATA *ch )
{
	return ch->ds + ch->stance/2;
}


//	short CvA1 for nonmagic armor
//	short CvA2 for magic
short get_cva( CHAR_DATA *victim )
{
	OBJ_DATA *armor;
	short adv = 0;
	
	armor = get_eq_char( victim, EQUIP_CHEST );

	if( armor != NULL )
	{
	if(  IS_OBJ_STAT(armor, ITEM_MAGIC ) )
	adv = armor_table[armor->value[3]].CvA2;
	else
	adv = armor_table[armor->value[3]].CvA1;
	}
	else
	adv = 25;

	armor = get_eq_char( victim, EQUIP_LEFTHAND );

	if( armor != NULL )
		adv -= 5;

	return adv;
}

short get_bolt_adv( CHAR_DATA *ch, CHAR_DATA *victim, int sn )
{
	OBJ_DATA *armor;
	short adv = 30;
	
	adv += spell_table[sn].level;

	armor = get_eq_char( victim, EQUIP_CHEST );
	
	if( armor != NULL && armor->item_type == ITEM_ARMOR )
	{
		if( armor_table[armor->value[3]].v0 == ARMOR_GROUP_LEATHER )
			adv -= 2;
		else if( armor_table[armor->value[3]].v0 == ARMOR_GROUP_SCALE )
			adv -= 4;
		else if( armor_table[armor->value[3]].v0 == ARMOR_GROUP_CHAIN )
			adv -= 6;
		else if( armor_table[armor->value[3]].v0 == ARMOR_GROUP_PLATE )
			adv -= 8;
	}

	return adv;
}


// returns success margin
short warding_check( CHAR_DATA *ch, CHAR_DATA *victim )
{
	char buf1[MAX_STRING_LENGTH];
	short total;
	short cs = get_cs( ch );
	short td = get_td( victim );
	short diceroll = number_range(1,100);
	short cva = get_cva( victim );
	
	total = cs-td+cva+diceroll;
	
	if( is_safe( ch, victim ) )
	return -1;
		
	// potency bonus to cs
	if( !IS_NPC(ch) )
	cs += stat_bonus(get_curr_Potency( victim ), victim, STAT_POTENCY);
	
	// willpower bonus to td
	if( !IS_NPC(victim) )
	td += stat_bonus(get_curr_Willpower( victim ), victim, STAT_WILLPOWER);

	sprintf( buf1, "   SPow: %s%d - MDef: %s%d + Adv. %d + roll: %s%d = %s%d",
	cs>-1 ? "+" : "", cs,
	td>-1 ? "+" : "", td,
	cva,
	diceroll>-1 ? "+" : "", diceroll,
	total>-1 ? "+" : "", total );

	act( buf1, ch, NULL, victim, TO_NOTVICT );
	act( buf1, ch, NULL, victim, TO_VICT );
    act( buf1, ch, NULL, victim, TO_CHAR );
	
	if( IS_NPC(victim) )
		victim->hunting = ch;
	
	if( total > 100 )
	{
	act( "   Warding failed!", ch, NULL, victim, TO_NOTVICT );
	act( "   Warding failed!", ch, NULL, victim, TO_VICT );
    act( "   Warding failed!", ch, NULL, victim, TO_CHAR );
	return total-100;
	}
	else
	{
	act( "   Warded off!", ch, NULL, victim, TO_NOTVICT );
	act( "   Warded off!", ch, NULL, victim, TO_VICT );
    act( "   Warded off!", ch, NULL, victim, TO_CHAR );
	return 0;
	}
}

short bolt_check( CHAR_DATA *ch, CHAR_DATA *victim, int sn )
{
	char buf1[MAX_STRING_LENGTH];
	short total;
	short as = get_bolt_as( ch );
	short ds = get_bolt_ds( victim );
	short adv = get_bolt_adv( ch, victim, sn );
	short diceroll = number_range(1,100);
	OBJ_DATA *wield;
	
	if( is_safe( ch, victim ) )
	return -1;
	
		// armor enchant bonus to ds
	
	wield = get_eq_char( victim, EQUIP_CHEST );
	if( wield != NULL && wield->item_type == ITEM_ARMOR && 
		wield->value[2] > -51 && wield->value[2] < 51)
		ds += wield->value[2];
	
	// shield enchant bonus to ds

	wield = get_eq_char( victim, EQUIP_LEFTHAND );
	if( wield != NULL && wield->item_type == ITEM_SHIELD && 
		wield->value[2] > -51 && wield->value[2] < 51)
		ds += 20 + wield->value[2];
	
	//spell aiming (skill) bonus	
		if( !IS_NPC(ch ) )
		as += skill_bonus(ch->pcdata->learned[gsn_spell_aiming]);
	
	if( victim->stun > 0 )
	ds -= 20;
	
	if( victim->position != POS_STANDING )
	ds -= 50;
	
	// speed bonus to ds
	if( !IS_NPC(victim) )
	ds += stat_bonus(get_curr_Speed( victim ), victim, STAT_SPEED );
	
	total = as-ds+adv+diceroll;

	sprintf( buf1, "   Aim: %s%d - BDef: %s%d + Adv. %d + roll: %s%d = %s%d",
	as>-1 ? "+" : "", as,
	ds>-1 ? "+" : "", ds,
	adv,
	diceroll>-1 ? "+" : "", diceroll,
	total>-1 ? "+" : "", total );

	act( buf1, ch, NULL, victim, TO_NOTVICT );
	act( buf1, ch, NULL, victim, TO_VICT );
    act( buf1, ch, NULL, victim, TO_CHAR );
	
	if( IS_NPC(victim) )
		victim->hunting = ch;
	
	if( total > 100 )
	{
	return total-100;
	}
	else
	{
	act( "   A clean miss.", ch, NULL, victim, TO_NOTVICT );
	act( "   A clean miss.", ch, NULL, victim, TO_VICT );
    act( "   A clean miss.", ch, NULL, victim, TO_CHAR );
	return 0;
	}
}

/*
	int i = number_range(1, 100);
	int j = 0;
	
	while (j 
	if (i > 95 )
	{
		j += 
		i += number_range( 1, 100);
	else if( i < 6 )
		i -= number_range( 1, 100);
	
	return i;
*/


// ok really need open for breakage

short open_1d100()
{
    int total = 0;
    int roll = 0;
    
    do 
    {
        roll = number_range(1, 100);
        total += roll;
    } while (roll > 95);
    
    return total;
}



//fixme open d100...encumberance...stats...armor...
bool man_check( CHAR_DATA *ch, CHAR_DATA *victim )
{
	char buf[MAX_STRING_LENGTH];
	short bonus = 0;
	short penalty = 0;
	short roll = open_1d100();
	
	bonus += ch->level*2;
	penalty += victim->level;
	
	if( !IS_NPC(ch) )
	{
		bonus += ch->pcdata->learned[gsn_combat_maneuvers]*2;
	}
	
	if( !IS_NPC(victim) )
	{
		penalty += victim->pcdata->learned[gsn_combat_maneuvers];
	}
	
	penalty += armor_penalty(victim); //negative
	
	sprintf( buf, "[Man %d vs. Def %d + open roll %d = %d]", bonus, penalty, roll, bonus+penalty+roll );
	act( buf, ch, NULL, NULL, TO_CHAR );
	act( buf, ch, NULL, NULL, TO_ROOM );
	
	if( bonus-penalty+roll > 100 )
		return TRUE;
	
	return FALSE;
}

void drop_hand( CHAR_DATA *ch, short loc )
{
	OBJ_DATA *obj = get_eq_char( ch, loc );
	
	if( obj != NULL )
	{
	act( "$n loses $s grip and $p falls to the ground.", ch, obj, NULL, TO_ROOM );
	act( "You lose your grip and $p falls to the ground.", ch, obj, NULL, TO_CHAR );;
	obj_from_char( obj );
	obj_to_room( obj, ch->in_room );
	}
}

void do_glance( CHAR_DATA *ch, char *argument )
{
	if( get_eq_char( ch, EQUIP_RIGHTHAND ) == NULL &&  get_eq_char( ch, EQUIP_LEFTHAND ) == NULL )
	{
		send_to_char( "You glance down at your empty hands.\r\n", ch );
		return;
	}
	else
		show_hands_to_char(ch,ch);
}

// this is only called if aggro or kill_aggro find a target
bool spell_attack( CHAR_DATA *mob, CHAR_DATA *victim )
{
		if( mob->predelay_info != NULL )
			return FALSE;
	
		//printf("called spell_attack: %s\r\n", mob->name);
	
        if ( mob->mana < 1)
                return FALSE;

        if ( mob->prepared_spell < 0 || spell_table[mob->prepared_spell].spell_fun == spell_null ) 
        {
			
			if( mob->class == CLASS_MAGE )
			{
                if( mob->level > 16 && mob->mana > 10  )
				{
				do_prepare( mob, "910" );
				do_cast( mob, victim->name );
				mob->wait = 3 * PULSE_PER_SECOND;
				return TRUE;
				}
				
				if( mob->level > 14 && mob->mana > 6  )
				{
				do_prepare( mob, "707" );
				do_cast( mob, victim->name );
				mob->wait = 3 * PULSE_PER_SECOND;
				return TRUE;
				}
			
			    if( mob->level > 12 && mob->mana > 5  )
				{
				do_prepare( mob, "906" );
				do_cast( mob, victim->name );
				mob->wait = 3 * PULSE_PER_SECOND;
				return TRUE;
				}
			
			
				if( mob->level > 9 && mob->mana > 2  )
				{
				do_prepare( mob, "903" );
				do_cast( mob, victim->name );
				mob->wait = 3 * PULSE_PER_SECOND;
				return TRUE;
				}
			
				if( mob->level > 0 && mob->mana > 0 )
				{
				do_prepare( mob, "901" );
				do_cast( mob, victim->name );
				mob->wait = 3 * PULSE_PER_SECOND;
				return TRUE;
				}
			}
			
			else if( mob->class == CLASS_PRIEST )
			{
                if( mob->level > 18 && mob->mana > 11  )
				{
				do_prepare( mob, "312" );
				do_cast( mob, victim->name );
				mob->wait = 3 * PULSE_PER_SECOND;
				return TRUE;
				}
				
				if( mob->level > 12 && mob->mana > 6  )
				{
				do_prepare( mob, "306" );
				do_cast( mob, victim->name );
				mob->wait = 3 * PULSE_PER_SECOND;
				return TRUE;
				}
				
				if( IS_SET( victim->affected_by, AFF_UNDEAD ) && mob->level > 9 && mob->mana > 3  )
				{
				do_prepare( mob, "303" );
				do_cast( mob, victim->name );
				mob->wait = 3 * PULSE_PER_SECOND;
				return TRUE;
				}
			}

        }
        else if ( spell_table[mob->prepared_spell].target == TAR_CHAR_OFFENSIVE )
        {
                do_cast( mob, victim->name );
				
				mob->wait = 3 * PULSE_PER_SECOND;
                return TRUE;
        }
        else
        {
                do_release( mob, "" );
                return FALSE;
        }

		multi_hit( mob, victim, TYPE_UNDEFINED );
        return TRUE; // eh it's an attack

}

bool spell_up( CHAR_DATA *mob )
{
        if ( mob->mana < 1)
            return FALSE;
			
		if( mob->predelay_info != NULL )
			return FALSE;

        if ( mob->prepared_spell < 0 || spell_table[mob->prepared_spell].spell_fun == spell_null ) 
        {
				if( mob->class == CLASS_MAGE )
				{
					
				if( mob->level > 13 && mob->mana > 13 && !is_affected(mob, gsn_elemental_defense_iii) )
				{
				do_prepare( mob, "414" );
					do_cast( mob, "" );
				mob->wait = 3 * PULSE_PER_SECOND;
				return TRUE;
				}
			
			    if( mob->level > 9 && mob->mana > 9 && !is_affected(mob, gsn_strength) )
				{
				do_prepare( mob, "509" );
					do_cast( mob, "" );
				mob->wait = 3 * PULSE_PER_SECOND;
				return TRUE;
				}
				
				if( mob->level > 5 && mob->mana > 5 && !is_affected(mob, gsn_elemental_defense_ii) )
				{
				do_prepare( mob, "406" );
					do_cast( mob, "" );
				mob->wait = 3 * PULSE_PER_SECOND;
				return TRUE;
				}
				
				if( mob->level > 2 &&  mob->mana > 2 && !is_affected(mob, gsn_protective_ward) )
				{
				do_prepare( mob, "503" );
					do_cast( mob, "" );
				mob->wait = 3 * PULSE_PER_SECOND;
				return TRUE;
				}
			
				if( mob->level > 0 && mob->mana > 0 && !is_affected(mob, gsn_elemental_defense_i) )
				{
				do_prepare( mob, "401" );
					do_cast( mob, "" );
				mob->wait = 3 * PULSE_PER_SECOND;
				return TRUE;
				}	
				
				}
				else if ( mob->class == CLASS_PRIEST )
				{
					
				if( mob->level > 19 && mob->mana > 19 && !is_affected(mob, gsn_lesser_shroud) )
				{
				do_prepare( mob, "120" );
					do_cast( mob, "" );
				mob->wait = 3 * PULSE_PER_SECOND;
				return TRUE;
				}

				if( mob->level > 18 && mob->mana > 18 && !is_affected(mob, gsn_spell_shield) )
				{	
				do_prepare( mob, "219" );
				do_cast( mob, "" );
				mob->wait = 3 * PULSE_PER_SECOND;
				return TRUE;
				}
					
			    if( mob->level > 6 && mob->mana > 6 && !is_affected(mob, gsn_spirit_warding_ii) )
				{
				do_prepare( mob, "107" );
				do_cast( mob, "" );
				mob->wait = 3 * PULSE_PER_SECOND;
				return TRUE;
				}
		
				if( mob->level > 0 && mob->mana > 0 && !is_affected(mob, gsn_spirit_warding_i) )
				{
				do_prepare( mob, "101" );
				do_cast( mob, "" );
				mob->wait = 3 * PULSE_PER_SECOND;
				return TRUE;
				}					
				}
        
        }

        else if ( spell_table[mob->prepared_spell].target == TAR_CHAR_DEFENSIVE
                || spell_table[mob->prepared_spell].target == TAR_CHAR_SELF )
        {
                do_cast( mob, "" );
				mob->wait = 3 * PULSE_PER_SECOND;
				mob->prepared_spell = 0;
                return TRUE;
        }
        else
        {
                do_release( mob, "" );
                return FALSE;
        }

        return FALSE;
}


bool aggro( CHAR_DATA *mob )
{
	CHAR_DATA *rch;

    for ( rch = mob->in_room->people; rch != NULL; rch = rch->next_in_room )
    {
	if ( rch == mob )
	    continue;
	
	if( rch->hit < 1 )
		continue;
	
	if (number_bits(2) == 0)
		continue;
	
	if( IS_NPC(rch) && IS_SET(rch->act, ACT_AGGRESSIVE ) ) //ironic, aggro mobs do not attack other aggro mobs.
		continue;
/*		
	if( IS_NPC(rch) &&  )
		printf("%s is aggro on %s and has passed the aggro flag check.\r\n", mob->short_descr, rch->short_descr );
*/
		
	if( !can_see( mob, rch ) )
		continue;
		
	if ( IS_SET( rch->affected_by, AFF_DEAD ) )
		continue;
	
	if ( IS_SET( mob->act, ACT_SPELLCASTER ) )
		spell_attack( mob, rch );
	else
		multi_hit( mob, rch, TYPE_UNDEFINED );
	
	return TRUE;
	}
	
	return FALSE;
}

bool kill_aggro( CHAR_DATA *mob )
{
	CHAR_DATA *rch;

    for ( rch = mob->in_room->people; rch != NULL; rch = rch->next_in_room )
    {
	if ( rch == mob )
	    continue;
	
	if( rch->hit < 1 )
		continue;
	
	if (number_bits(2) == 0)
		continue;
	
	if( !IS_NPC(rch) )
		continue;
	if( !can_see( mob, rch ) )
		continue;
		
	if ( IS_SET( rch->affected_by, AFF_DEAD ) )
		continue;
		
	if ( !IS_SET( rch->act, ACT_AGGRESSIVE ) )
		continue;
		
	if ( IS_SET( mob->act, ACT_SPELLCASTER ) )
		spell_attack( mob, rch );
	else
		multi_hit( mob, rch, TYPE_UNDEFINED );
	
	return TRUE;
	}
	
	return FALSE;
}

bool is_valid_portal( OBJ_DATA *obj )
{
	if( obj == NULL )
	return FALSE;
	
	if ( obj->item_type != ITEM_PORTAL && obj->item_type != ITEM_BUILDING && obj->item_type != ITEM_DOOR && obj->item_type != ITEM_CLIMB && obj->item_type != ITEM_SWIM)
	return FALSE;
			
	if ( get_room_index( obj->value[0] ) == NULL )
	return FALSE;

	return TRUE;
}

bool wander( CHAR_DATA *ch )
{
    EXIT_DATA *pexit;
    int door;
	
	if(number_bits(2) != 0)
		return FALSE;

	if ( ( door = number_range(0,20) ) <= 10
	&& ( pexit = ch->in_room->exit[door] ) != NULL
	&&   pexit->to_room != NULL
	&&   !IS_SET(pexit->exit_info, EX_CLOSED)
	&&   !IS_SET(pexit->to_room->room_flags, ROOM_NO_MOB)
	&& ( !IS_SET(ch->act, ACT_STAY_AREA)
	||   pexit->to_room->area == ch->in_room->area ) )
	{
	    move_char( ch, door );
		return TRUE;
	}
	
	if ( ch->in_room->contents != NULL )
	{
	    OBJ_DATA *obj;

	    for ( obj = ch->in_room->contents; obj; obj = obj->next_content )
	    {						
			if ( is_valid_portal( obj ) && number_range(0,20) == 20 )
			{
			ROOM_INDEX_DATA *room = get_room_index( obj->value[0] );
				
			if( !(IS_SET(ch->act, ACT_STAY_AREA) && room->area != ch->in_room->area) )
			{
			do_enter( ch, obj->name );
			return TRUE;
			}
			}
		}
	}
	
	return FALSE;
}


#if 0
bool scavenge( CHAR_DATA *ch )
{
	if ( ch->in_room->contents != NULL
	&&   number_bits( 2 ) == 0 )
	{
	    OBJ_DATA *obj;
	    OBJ_DATA *obj_best;
	    int max;

	    max         = 1;
	    obj_best    = 0;
	    for ( obj = ch->in_room->contents; obj; obj = obj->next_content )
	    {
		if ( CAN_WEAR(obj, ITEM_EQUIP_TAKE) && (obj->item_type == ITEM_WEAPON || obj->item_type == ITEM_SHIELD || obj->item_type == ITEM_ARMOR) && obj->cost > max )
		{
		    obj_best    = obj;
		    max         = obj->cost;
		}
	    }
		
		/*
		if( get_eq_char( ch, EQUIP_RIGHTHAND ) != NULL &&  get_eq_char( ch, EQUIP_LEFTHAND ) != NULL )
		return FALSE;

	    if ( obj_best )
	    {
		
		if( get_eq_char( ch, EQUIP_RIGHTHAND ) == NULL || get_eq_char( ch, EQUIP_LEFTHAND ) == NULL )
		{
		obj_from_room( obj_best );
		obj_to_char( obj_best, ch );
		obj_to_free_hand( obj_best, ch );
		act( "$n picks up $p.", ch, obj_best, NULL, TO_ROOM );
		
		if( get_eq_char( ch, EQUIP_RIGHTHAND ) == NULL && obj_best && obj_best->item_type == ITEM_WEAPON )
		{
				do_swap( ch, "" );
				return TRUE;
		}	
		
		if( obj->item_type == ITEM_ARMOR)
		{		
				do_wear(ch, obj_best->name);
				return TRUE;
		}			

		return TRUE;
		
		}
	    }
		*/
	}
	
	return FALSE;
}
#endif

bool scavenge( CHAR_DATA *ch )
{
	if( get_eq_char( ch, EQUIP_RIGHTHAND ) != NULL && get_eq_char( ch, EQUIP_LEFTHAND ) != NULL )
		return FALSE;
	
	if (  ch->in_room->contents != NULL
	&&   number_bits( 2 ) == 0 )
	{
	    OBJ_DATA *obj;
	    OBJ_DATA *obj_best;
	    int max;

	    max         = 1;
	    obj_best    = 0;
	    for ( obj = ch->in_room->contents; obj; obj = obj->next_content )
	    {
		//if ( get_eq_char( ch, EQUIP_CHEST ) != NULL && //obj_best->item_type != ITEM_ARMOR  )
		//{
		//continue;
		//}
		
		if ( CAN_WEAR(obj, ITEM_EQUIP_TAKE) && obj->cost > max )
		{
		    obj_best    = obj;
		    max         = obj->cost;
		}
	    }

	    if ( obj_best )
	    {
			
		obj_from_room( obj_best );
		obj_to_char( obj_best, ch );
		obj_to_free_hand( obj_best, ch );
		act( "$n picks up $p.", ch, obj_best, NULL, TO_ROOM );
		
		//fixme - swap
		//do_get(ch, obj_best->name);
		
		
		if( obj_best->item_type == ITEM_ARMOR)
		{		
				do_wear(ch, obj_best->name);
				return TRUE;
		}		

		//if( get_eq_char( ch, EQUIP_RIGHTHAND ) == NULL && //obj_best && obj_best->item_type == ITEM_WEAPON )
		//{
		//		do_swap( ch, "" );
		//		return TRUE;
		//}	
		
		return TRUE;
	    }
	}
	
	return FALSE;
}

void do_skills( CHAR_DATA *ch, char *argument )
{
	short i;
	int col;
	
	send_to_char( "ALL IMPLEMENTED SKILLS:\r\n", ch);
	
	for (i = 0; i < MAX_SKILL; i++ )
	{
	//printf_to_char( ch, "%s %s\r\n", skill_table[i].name, skill_table[i].type == SKILL_NOT_IMPLEMENTED ? "(not implemented)" : "" );
	
	if( skill_table[i].type != SKILL_NOT_IMPLEMENTED )
	printf_to_char( ch, "%21s", skill_table[i].name );

	if ( ++col % 2 == 0 )
	send_to_char( "\n\r", ch );
	}
}

void do_practice( CHAR_DATA *ch, char *argument )
{
    int sn, pcost, mcost;
 
    if ( IS_NPC(ch) )
	return;
 
    if ( argument[0] == '\0' )
    {
	for ( sn = 0; sn < MAX_SKILL; sn++ )
	{
	    if ( skill_table[sn].name == NULL )
		break;
	
	if( ch->pcdata->learned[sn]+1 > ch->level )
	{
	pcost = skill_table[sn].basePTPs*2;
	mcost = skill_table[sn].baseMTPs*2;
	}
	else
	{
	pcost = skill_table[sn].basePTPs;
	mcost = skill_table[sn].baseMTPs;
	}
 
		if( skill_table[sn].type != SKILL_NOT_IMPLEMENTED && ch->pcdata->learned[sn]  )
		{
 	    printf_to_char( ch, "%20s%s - %3d ranks (%3d bonus) - Next rank: (%2d/%2d)\r\n",
	    //printf_to_char( ch, "%20s%s - %3d ranks (%3d bonus) - Next rank: (%2d/%2d)\r\n",
		skill_table[sn].name, 
		skill_table[sn].type == SKILL_NOT_IMPLEMENTED ? "*" : " ",
		ch->pcdata->learned[sn], 
		skill_bonus(ch->pcdata->learned[sn]),
		pcost,
		mcost );
		}

	}
 
	//if ( col % 3 != 0 )
	    //send_to_char( "\n\r", ch );
 
 //Skills marked with an asterisk (*) are not yet implemented
 
	  printf_to_char(ch, "\r\nType SKILLS to see all possible skills.\r\n\r\nYou have %d physical training points and %d mental training points.\r\n",
	    		ch->pcdata->PTPs, ch->pcdata->MTPs );
    }
    else
{
      if( ( sn = skill_lookup( argument ) ) < 0 )
      {
         send_to_char( "No such skill.\r\n", ch );
         return;
      }
	  
	  if( skill_table[sn].type == SKILL_NOT_IMPLEMENTED  )
      {
         send_to_char( "That skill is not implemented yet, so do not train it.\r\n", ch );
         return;
      }
	  
	  //white mana
	  if( sn == gsn_elemental_mana_control && ch->pcdata->learned[gsn_spirit_mana_control] > 0 )
	  {
         send_to_char( "You have already started learning white magic and cannot risk corruption.\r\n", ch );
         return;
      }	  
	  
	  //black mana
	  if( sn == gsn_spirit_mana_control && ch->pcdata->learned[gsn_elemental_mana_control] > 0 )
	  {
         send_to_char( "You have already begun down the path of black magic, which eschews such weakness.\r\n", ch );
         return;
      }
	  
	  //white wiz
	  if( sn == gsn_elemental_lore && ch->pcdata->learned[gsn_spiritual_lore] > 0 )
	  {
         send_to_char( "You have already started learning white magic and cannot risk corruption.\r\n", ch );
         return;
      }	  
	  
	  //black wiz
	  if( sn == gsn_spiritual_lore && ch->pcdata->learned[gsn_elemental_lore] > 0 )
	  {
         send_to_char( "You have already begun down the path of black magic, which eschews such weakness.\r\n", ch );
         return;
      }
	  
	if( ch->pcdata->learned[sn]+1 > ch->level )
	{
	pcost = skill_table[sn].basePTPs*2;
	mcost = skill_table[sn].baseMTPs*2;
	}
	else
	{
	pcost = skill_table[sn].basePTPs;
	mcost = skill_table[sn].baseMTPs;
	}

		if ( ch->pcdata->PTPs < pcost )
		{
	    printf_to_char( ch, "You need more PTPs to train in this skill.\r\n" );
		
		//doubling mtps
		mcost = mcost + (( pcost - ch->pcdata->PTPs )*2);
		pcost = ch->pcdata->PTPs;
		
		if( mcost > ch->pcdata->MTPs )
		{
			    printf_to_char( ch, "Need %d total MTPs to bypass and overtrain.\r\n", mcost );
				return;
		}
		else
		{
			printf_to_char( ch, "Spending %d total MTPs to bypass and overtrain...\r\n", mcost );
		}
	}
	
	if ( ch->pcdata->MTPs < mcost )
	{
	    printf_to_char( ch, "You need more MTPs to train in this skill.\r\n" );
		
		//doubling ptps
		pcost = pcost + (( mcost - ch->pcdata->MTPs )*2);
		mcost = ch->pcdata->MTPs;
		
		if( pcost > ch->pcdata->PTPs )
		{
			    printf_to_char( ch, "Need %d total PTPs to bypass and overtrain.\r\n", pcost );
				return;
		}
		else
		{
			printf_to_char( ch, "Spending %d total PTPs to bypass and overtrain...\r\n", pcost );
		}
	}
	
	switch( skill_table[sn].difficulty )
	{
	case	DIFFICULTY_TRIVIAL:
	case	DIFFICULTY_EASY:
	if( ch->pcdata->learned[sn]+1 > ch->level * 3 )
	{
	    printf_to_char( ch, "You cannot train in this skill more than three times per level.\r\n" );
	    return;
	}
	
	case 	DIFFICULTY_HARD:
	case 	DIFFICULTY_ARDUOUS:
	if( ch->pcdata->learned[sn]+1 > ch->level  )
	{
	    printf_to_char( ch, "You cannot train in this skill more than once per level.\r\n" );
	    return;
	}
	
	default:
	case 	DIFFICULTY_NORMAL:
	if( ch->pcdata->learned[sn]+1 > ch->level * 2 )
	{
	    printf_to_char( ch, "You cannot train in this skill more than two times per level.\r\n" );
	    return;
	}
	}

		ch->pcdata->PTPs = ch->pcdata->PTPs - pcost;
		ch->pcdata->MTPs = ch->pcdata->MTPs - mcost;

		printf_to_char( ch, "You use %d PTPs and %d MTPs to gain a rank of %s.\r\n", pcost, mcost,
			skill_table[sn].name );

	    ch->pcdata->learned[sn]++;
		
		//fixme in case stat bonus increases...
		if( sn == gsn_physical_fitness )
		{
			if( ch->pcdata->learned[gsn_physical_fitness]-1 < ch->level )
			{
				ch->max_hit += race_table[ch->race].HP_per_level;
				
				if( ch->max_hit > race_table[ch->race].max_Health )
					ch->max_hit = race_table[ch->race].max_Health;
				
				printf_to_char(ch, "Your maximum health is now %d.\r\n", ch->max_hit );
			}				
			else
			{
				ch->max_hit += 1;
				
				if( ch->max_hit > race_table[ch->race].max_Health )
					ch->max_hit = race_table[ch->race].max_Health;
				
				printf_to_char(ch, "Your maximum health is now %d.\r\n", ch->max_hit );
			}	
			return;
		}
		if( sn == gsn_harness_power)
		{
			if( ch->pcdata->learned[gsn_harness_power]-1 < ch->level )
			{
				ch->max_mana += 3;
				printf_to_char(ch, "Your maximum mana is now %d.\r\n", ch->max_mana );
			}				
			else
			{
				ch->max_mana += 1;
				printf_to_char(ch, "Your maximum mana is now %d.\r\n", ch->max_mana );
			}		

			return;
		}
   }
    return;
}


void wound( CHAR_DATA *ch, short loc, short severity, short bleed, bool silent )
{
	if( ch->body[loc] == 0 )
		ch->body[loc] = severity;
		
	else if( ch->body[loc] == 1 )
	{
		if( severity == WOUND_RANK_1 )
			ch->body[loc] = 2;
		else if( severity == WOUND_RANK_2 )
			ch->body[loc] = 2;
		else if( severity == WOUND_RANK_3 )
			ch->body[loc] = 3;
	}
	
	else if( ch->body[loc] == 2 )
	{
		if( severity == WOUND_RANK_1 )
			ch->body[loc] = 2;
		else if( severity == WOUND_RANK_2 )
			ch->body[loc] = 3;
		else if( severity == WOUND_RANK_3 )
			ch->body[loc] = 3;
	}
	
	else if( ch->body[loc] == 3 )
	{
		ch->body[loc] = 3;
	}
	
	if( ch->hit > 0 && severity > WOUND_RANK_1 )
	{
		if( ch->position != POS_PRONE )
		{
		if( loc == BODY_R_LEG || loc == BODY_L_LEG || loc == BODY_R_FOOT || loc == BODY_L_FOOT )
		{
		if( !silent )
		{
		act( "  $n is knocked down!", ch, NULL, NULL, TO_ROOM );
		act( "  You are knocked down!", ch, NULL, NULL, TO_CHAR );
		}
		ch->position = POS_PRONE;
		}
		}
		
		if( loc == BODY_R_ARM || loc == BODY_R_HAND ) 
			drop_hand( ch, EQUIP_RIGHTHAND );
		
		if( loc == BODY_L_ARM || loc == BODY_L_HAND ) 
			drop_hand( ch, EQUIP_LEFTHAND );
			
	}
	
	if( !IS_DEAD(ch) && bleed > BLEED_NONE && !IS_SET( ch->affected_by, AFF_UNDEAD ) && !IS_SET( ch->affected_by, AFF_NONCORP ) )
	{
		if( !silent )
		{
		if( ch->bleed[loc] == 0 )
		{
		act( "   $n starts bleeding!", ch, NULL, NULL, TO_ROOM );
		printf_to_char( ch, "   Your %s starts bleeding at %d per!\r\n", body_name[loc], bleed );
		}
		else
		{
		act( "   $n's bleeding gets worse!", ch, NULL, NULL, TO_ROOM );
		printf_to_char( ch, "   Your %s is now bleeding at %d per!\r\n", body_name[loc], bleed + ch->bleed[loc] );
		}
		}
		
		ch->bleed[loc] += bleed;
	}
}



OBJ_DATA *create_random_gem( int level )
{
	char buf[MSL];
	OBJ_DATA *gem = create_object( get_obj_index( OBJ_VNUM_GEM ), 0 );
	int num;
	
	num = number_range(0,level);
	
	if( num > MAX_GEM-1 )
		num = MAX_GEM-1;
	
	strcpy( buf, "" );
	
	if( gem_table[num].name != NULL )
	{
	sprintf(buf, "%s", gem_table[num].name );	
	gem->name = str_dup(buf);
	
	strcpy( buf, "" );
	sprintf(buf, "%s %s", select_a_an( gem_table[num].name ), gem_table[num].name );	
	gem->short_descr = str_dup( buf );
	
	gem->value[0] = gem_table[num].tier;
	gem->cost = number_range(gem_table[num].min, gem_table[num].max);
	}
	else
	{
	gem->name = str_dup("piece painted glass");
	gem->short_descr = str_dup("a piece of painted glass");
	
	gem->value[0] = 0;
	gem->cost = 0;
	}	

	return gem;
}


void enchant_shield( OBJ_DATA *weapon, int enchant )
{
		AFFECT_DATA *paf;
		paf			= alloc_perm( sizeof(*paf) );
		paf->type		= -1;
		paf->duration		= -1;
		
		if( enchant < 1 )
			return;
		
		SET_BIT( weapon->extra_flags, ITEM_MAGIC );
		
		if( enchant > 5 )
		enchant = 5;
		
		paf->modifier = enchant;
		
		//fixme capped by magic metal
		if( dice(1,100) > 50 )
		{
		paf->location		= APPLY_DS;
		paf->modifier		= enchant * 5;
		}
		else
		{
		switch ( number_range(0,9 ) )
		{
			case 0: paf->location = APPLY_STR; paf->modifier		= enchant*2; break;
			case 1: paf->location = APPLY_END; paf->modifier		= enchant*2; break;
			case 2: paf->location = APPLY_DEX; paf->modifier		= enchant*2; break;
			case 3: paf->location = APPLY_SPD; paf->modifier		= enchant*2; break;
			case 4: paf->location = APPLY_WIL; paf->modifier		= enchant*2; break;
			case 5: paf->location = APPLY_POT; paf->modifier		= enchant*2; break;
			case 6: paf->location = APPLY_JDG; paf->modifier		= enchant*2; break;
			case 7: paf->location = APPLY_WIS; paf->modifier		= enchant*2; break;
			case 8: paf->location = APPLY_CHA; paf->modifier		= enchant*2; break;
			default:
			case 9: paf->location = APPLY_INT; paf->modifier		= enchant*2; break;
		}
		}
		paf->bitvector		= 0;
		paf->next		= weapon->affected;
		weapon->affected	= paf;
}

void enchant_weapon( OBJ_DATA *weapon, int enchant )
{
		AFFECT_DATA *paf;
		paf			= alloc_perm( sizeof(*paf) );
		paf->type		= -1;
		paf->duration		= -1;
		
		if( enchant < 1 )
			return;
		
		SET_BIT( weapon->extra_flags, ITEM_MAGIC );
		
		if( enchant > 5 )
		enchant = 5;
		
		paf->modifier = enchant;
		
		//fixme capped by magic metal
		if( dice(1,100) > 50 )
		{
		paf->location		= APPLY_AS;
		paf->modifier		= enchant * 5;
		}
		else
		{
		switch ( number_range(0,9 ) )
		{
			case 0: paf->location = APPLY_STR; paf->modifier		= enchant*2; break;
			case 1: paf->location = APPLY_END; paf->modifier		= enchant*2; break;
			case 2: paf->location = APPLY_DEX; paf->modifier		= enchant*2; break;
			case 3: paf->location = APPLY_SPD; paf->modifier		= enchant*2; break;
			case 4: paf->location = APPLY_WIL; paf->modifier		= enchant*2; break;
			case 5: paf->location = APPLY_POT; paf->modifier		= enchant*2; break;
			case 6: paf->location = APPLY_JDG; paf->modifier		= enchant*2; break;
			case 7: paf->location = APPLY_WIS; paf->modifier		= enchant*2; break;
			case 8: paf->location = APPLY_CHA; paf->modifier		= enchant*2; break;
			default:
			case 9: paf->location = APPLY_INT; paf->modifier		= enchant*2; break;
		}
		}
		paf->bitvector		= 0;
		paf->next		= weapon->affected;
		weapon->affected	= paf;
}

void enchant_armor( OBJ_DATA *armor, int enchant )
{
		AFFECT_DATA *paf;
		paf			= alloc_perm( sizeof(*paf) );
		paf->type		= -1;
		paf->duration		= -1;
		
		if( enchant < 1 )
			return;
		
		SET_BIT( armor->extra_flags, ITEM_MAGIC );
		
		if( enchant > 6 )
		enchant = 6;
		
		//fixme capped by magic metal
		if( dice(1,100) > 75 ) // 25 percent of armor is enchanted.
		{
		paf->location		= APPLY_DS;
		paf->modifier		= enchant * 5;
		}
		else
		{
		switch ( number_range(0,9 ) )
		{
			case 0: paf->location = APPLY_STR; paf->modifier		= enchant*2; break;
			case 1: paf->location = APPLY_END; paf->modifier		= enchant*2; break;
			case 2: paf->location = APPLY_DEX; paf->modifier		= enchant*2; break;
			case 3: paf->location = APPLY_SPD; paf->modifier		= enchant*2; break;
			case 4: paf->location = APPLY_WIL; paf->modifier		= enchant*2; break;
			case 5: paf->location = APPLY_POT; paf->modifier		= enchant*2; break;
			case 6: paf->location = APPLY_JDG; paf->modifier		= enchant*2; break;
			case 7: paf->location = APPLY_WIS; paf->modifier		= enchant*2; break;
			case 8: paf->location = APPLY_CHA; paf->modifier		= enchant*2; break;
			default:
			case 9: paf->location = APPLY_INT; paf->modifier		= enchant*2; break;
		}
		}
		
		
		paf->bitvector		= 0;
		paf->next		= armor->affected;
		armor->affected	= paf;
}

OBJ_DATA *create_random_armor( int level  )
{
	char buf[MSL];
	int matl;
	bool metal = FALSE;
	OBJ_DATA *armor = create_object( get_obj_index( OBJ_VNUM_ARMOR ), 0 );
	int num = number_range(1,MAX_ARMOR-1);
	
	if( level < 5 )
	matl = number_range(0,2);
	else if( level < 10 )
	matl = number_range(0,3);
	else if( level < 15 )
	matl = number_range(0,4);
	else if( level < 20 )
	matl = number_range(0,5);
	else if( level < 25 )
	matl = number_range(0,6);
	else
	matl = number_range(0,7);
		
	strcpy( buf, "" );
	
	if( num < 8 )
	{
	sprintf(buf, "%s", armor_table[num].name );	
	armor->name = str_dup(buf);
	}
	else
	{
	sprintf(buf, "%s %s", armor_table[num].name, metal_table[matl].name );	
	armor->name = str_dup(buf);
	}
	
	strcpy( buf, "" );
	
	if( num < 8 )
	{
	sprintf(buf, "some %s",  armor_table[num].name );	
	armor->short_descr = str_dup( buf );
	}	
	else if( num < 21 )
	{
		metal = TRUE;
	sprintf(buf, "some %s %s",  metal_table[matl].name, armor_table[num].name );	
	armor->short_descr = str_dup( buf );
	}
	else
	{
		metal = TRUE;
	sprintf(buf, "%s %s %s", select_a_an(  metal_table[matl].name ),  metal_table[matl].name, armor_table[num].name );	
	armor->short_descr = str_dup( buf );
	}
	
	armor->wear_flags = armor_table[num].worn;
	
	armor->value[0] = armor_table[num].v0;
	armor->value[1] = armor_table[num].v1;
	armor->value[2] = armor_table[num].v2;
	armor->value[3] = armor_table[num].v3;
	
	armor->st = armor_table[num].st;
	
	if(metal)
	armor->st += metal_table[matl].st_modifier;
	
	armor->du = armor_table[num].du;
	
	if(metal)
	armor->du += metal_table[matl].du_modifier;
	
	if( num > 7 )
		armor->value[2] += metal_table[matl].enchant;
	 
	armor->value[3] = armor_table[num].v3;
				
	armor->weight = armor_table[num].weight;
	
	if( num > 7 )
		armor->weight = armor->weight * metal_table[matl].weight_modifier / 100;
	
	armor->cost = armor_table[num].cost;
	
	if( num > 7 )
		armor->cost *= metal_table[matl].cost_modifier;
	
	if( number_range(1,100) == 100 )
	enchant_armor( armor, number_range(0,level/6) );

	return armor;
}



OBJ_DATA *create_random_weapon( int level )
{
	char buf[MSL];
	OBJ_DATA *weapon = create_object( get_obj_index( OBJ_VNUM_WEAPON ), 0 );
	int num = number_range(1,19); //no closed fist, no natural weapons, careful here
	int matl;
	
	if( level < 5 )
	matl = number_range(0,2);
	else if( level < 10 )
	matl = number_range(0,3);
	else if( level < 15 )
	matl = number_range(0,4);
	else if( level < 20 )
	matl = number_range(0,5);
	else if( level < 25 )
	matl = number_range(0,6);
	else
	matl = number_range(0,7);
		
	strcpy( buf, "" );
	
	sprintf(buf, "%s %s", weapon_table[num].name, metal_table[matl].name );	
	weapon->name = str_dup(buf);
	
	strcpy( buf, "" );
	
	sprintf(buf, "%s %s %s", select_a_an(  metal_table[matl].name ),  metal_table[matl].name, weapon_table[num].name );	
	weapon->short_descr = str_dup( buf );
	
	weapon->value[0] = num;
	weapon->value[1] = 0;
	weapon->value[2] = metal_table[matl].enchant;
	weapon->value[3] = 0;
	
	weapon->st = weapon_table[num].st;
	
	weapon->st += metal_table[matl].st_modifier;
	
	weapon->du = weapon_table[num].du;
	
	weapon->du += metal_table[matl].du_modifier;
	
	weapon->weight = weapon_table[num].weight * metal_table[matl].weight_modifier / 100;
	weapon->cost = weapon_table[num].cost * metal_table[matl].cost_modifier;

	if( number_range(1,50) == 50 )
	enchant_weapon( weapon, number_range(0,level/6) );
	
	return weapon;
}



OBJ_DATA *create_random_shield( int level )
{
	char buf[MSL];
	OBJ_DATA *shield = create_object( get_obj_index( OBJ_VNUM_SHIELD ), 0 );
	int num = number_range(0,MAX_SHIELD-1);
	int matl;
	
	if( level < 5 )
	matl = number_range(0,2);
	else if( level < 10 )
	matl = number_range(0,3);
	else if( level < 15 )
	matl = number_range(0,4);
	else if( level < 20 )
	matl = number_range(0,5);
	else if( level < 25 )
	matl = number_range(0,6);
	else
	matl = number_range(0,7);
		
	strcpy( buf, "" );
	
	sprintf(buf, "%s %s", shield_table[num].name, metal_table[matl].name );	
	shield->name = str_dup(buf);
	
	strcpy( buf, "" );
	
	sprintf(buf, "%s %s %s", select_a_an(  metal_table[matl].name ),  metal_table[matl].name, 
	shield_table[num].name );	
	shield->short_descr = str_dup( buf );
	
	shield->value[0] = num;
	shield->value[1] = shield_table[num].size;
	shield->value[2] = metal_table[matl].enchant;
	shield->value[3] = 0;
	
	shield->st = shield_table[num].st;
	
	shield->st += metal_table[matl].st_modifier;
	
	shield->du = shield_table[num].du;
	
	shield->du += metal_table[matl].du_modifier;
	
	shield->weight = shield_table[num].weight * metal_table[matl].weight_modifier / 100;
	
	//fixme cost in table?
	shield->cost = ( (shield_table[num].size+1) * 50 ) * metal_table[matl].cost_modifier;

	if( number_range(1,75) == 75 )
	enchant_shield( shield, number_range(0,level/6) );
	
	return shield;
}

bool add_trap (OBJ_DATA *obj, int level)
{
 AFFECT_DATA *paf;

   if ( affect_free == NULL )
    {
	paf		= alloc_perm( sizeof(*paf) );
    }
    else
    {
	paf		= affect_free;
	affect_free	= affect_free->next;
    }

    paf->type		= -1;
    paf->duration	= -1;
   
    paf->location	= APPLY_NONE;
    paf->modifier	= level * number_range(2,6);

    paf->bitvector	= AFF_TRAPPED;
    paf->next		= obj->affected;
    obj->affected	= paf;

    SET_BIT( obj->extra_flags, ITEM_TRAPPED );
    return TRUE;
}



OBJ_DATA *create_random_box( int level )
{
    //OBJ_DATA *obj;
	OBJ_INDEX_DATA *pBoxIndex;
    OBJ_DATA *box;
	OBJ_DATA *obj;
	char buf[MSL];
    int a;
	int m;
	int n;
	
	pBoxIndex = get_obj_index(44);

    box = create_object( pBoxIndex, 0 );
	
	
	a = number_range(0,MAX_BOX_ADJ-1);
	m = number_range(0,MAX_BOX_MATL-1);
	n = number_range(0,MAX_BOX_NOUN-1);
	
	
	sprintf(buf, "%s %s %s", box_adj_table[a].name, 
							 box_matl_table[m].name, 
							 box_noun_table[n].name );
							 
	box->name = str_dup( buf );
	
	sprintf(buf, "%s %s", select_a_an(box->name), box->name );
	
	box->short_descr = str_dup( buf );
	
	//always locked
    SET_BIT( box->value[1], CONT_CLOSEABLE );
    SET_BIT( box->value[1], CONT_CLOSED );
    SET_BIT( box->value[1], CONT_LOCKED );
	
	box->weight = number_range(9,11); // each box weighs between 9-11 and
	box->value[0] = 100; // holds 100
	box->cost = dice(4,5); // and is worth...almost...nothing.
	
	box->value[3] = number_range(5,25) * level * -1;
	
	box->description = str_dup( "It looks like the perfect place to store valuables.\r\n" );
	
	SET_BIT(box->wear_flags, ITEM_EQUIP_TAKE);
	
	

    if( dice(1,100) > 1 ) // ALMOST ALWAYS COINS
    {
    int amount = number_range( level+1, (level+1) * 10);

    obj = create_money( amount );
    if( obj != NULL )
    obj_to_obj( obj, box );
    }

    if( dice(1,100) > 30 ) // USUALLY A GEM
    {
    obj = create_random_gem( level );
    if( obj != NULL )
    obj_to_obj( obj, box );

    }

	
	// WEAPONS AND ARMOR ARE RARE

	
    if( dice(1,100) > 95 )
    {
    switch( dice(1,6) )
	{
		case 1:
		case 2:
		case 3:
		obj = create_random_weapon( level );
		break;
		case 4:
		obj = create_random_shield( level );
		break;
		case 5:
		case 6:
		default:
		obj = create_random_armor( level );
		break;
    }
	if( obj != NULL )
    obj_to_obj( obj, box );
	}
	
	if( number_range(1,3) == 1 )  // 33.3r% boxes are trapped
		add_trap( box, level );
		
	if( is_name( obj->name, "mithril" ) )
		obj->cost = number_range(1000,2000);
	else if( is_name( obj->name, "gold" ) )
		obj->cost = number_range(400,600);
	else if( is_name( obj->name, "silver" ) )
		obj->cost = number_range(200,300);
	else
		obj->cost = number_fuzzy(100);
	
    return box;
}




OBJ_DATA *create_random_container( int level )
{
    //OBJ_DATA *obj;
	OBJ_INDEX_DATA *pContainerIndex;
    OBJ_DATA *container;
	char buf[MSL];
    int a;
	int m;
	int n;
	
	pContainerIndex = get_obj_index(46);

    container = create_object( pContainerIndex, 0 );
	
	
	a = number_range(0,MAX_CONTAINER_ADJ-1);
	m = number_range(0,MAX_CONTAINER_MATL-1);
	n = number_range(0,MAX_CONTAINER_NOUN-1);
	
	
	sprintf(buf, "%s %s %s", container_adj_table[a].name, 
							 container_matl_table[m].name, 
							 container_noun_table[n].name );
							 
	container->name = str_dup( buf );
	
	sprintf(buf, "%s %s", select_a_an(container->name), container->name );
	
	container->short_descr = str_dup( buf );
	
    SET_BIT( container->value[1], CONT_CLOSEABLE );
	
	container->weight = number_range(9,11); // fixme
	container->value[0] = 100; // fixme
	container->cost = dice(4,5); // fixme
	
	container->value[3] = 0;
	
	//box->description = str_dup( "It looks like the perfect place to store valuables.\r\n" );
	
	SET_BIT(container->wear_flags, ITEM_EQUIP_TAKE|container_noun_table[n].flags);

    return container;
}





/*
 * Return true if a char is delayed by a skill.
 */
bool is_delayed( CHAR_DATA *ch, int sn )
{
    SKILL_DELAY_DATA *delay;

    for ( delay = ch->pcdata->skill_delays; delay != NULL; delay = delay->next )
    {
	if ( delay->skill == sn )
	    return TRUE;
    }

    return FALSE;
}

/*
 * Give a delay to a char.
 */
void delay_to_char( CHAR_DATA *ch, SKILL_DELAY_DATA *delay )
{
    SKILL_DELAY_DATA *delay_new;

    if ( skill_delay_free == NULL )
    {
	delay_new	= alloc_perm( sizeof(*delay_new) );
    }
    else
    {
	delay_new		= skill_delay_free;
	skill_delay_free	= skill_delay_free->next;
    }

    *delay_new		= *delay;
    delay_new->next	= ch->pcdata->skill_delays;
    ch->pcdata->skill_delays	= delay_new;

    return;
}

/*
 * Remove a delay from a char.
 */
void delay_remove( CHAR_DATA *ch, SKILL_DELAY_DATA *delay )
{
    if ( ch->pcdata->skill_delays == NULL )
    {
	bug( "Delay_remove: no delay.", 0 );
	return;
    }

    if ( delay == ch->pcdata->skill_delays )
    {
	ch->pcdata->skill_delays	= delay->next;
    }
    else
    {
	SKILL_DELAY_DATA *prev;

	for ( prev = ch->pcdata->skill_delays; prev != NULL; prev = prev->next )
	{
	    if ( prev->next == delay )
	    {
		prev->next = delay->next;
		break;
	    }
	}

	if ( prev == NULL )
	{
	    bug( "Delay_remove: cannot find delay.", 0 );
	    return;
	}
    }

    delay->next	= skill_delay_free;
    skill_delay_free	= delay;
    return;
}


void skill_improve( CHAR_DATA *ch, int sn )
{
    SKILL_DELAY_DATA delay;
    int iMax;
	int diff = 0;

    iMax = 400;

    if (IS_NPC(ch))
	return;

    if ( is_delayed( ch, sn ) )
	return;

/* implement delay between improves */
	
	if( skill_table[sn].difficulty == DIFFICULTY_TRIVIAL || DIFFICULTY_EASY )
		diff = ch->level * 3;
	else if( skill_table[sn].difficulty == DIFFICULTY_HARD || DIFFICULTY_ARDUOUS )
		diff = ch->level * 1;
	else
		diff = ch->level * 2;
	
	iMax = skill_bonus(diff);
	
    iMax = UMIN( iMax, 400 );

    if (dice(1,iMax) < skill_bonus(ch->pcdata->learned[sn]))
    {
/* learned nothing */
	delay.skill = sn;
	delay.duration = (10 / diff) * 75 / (get_curr_Intelligence( ch )*2);
	delay_to_char( ch, &delay );
	return;
    }

    ch->pcdata->learned[sn]++; // only go up 1

/* delay to next possible learn */
    delay.skill = sn;
    delay.duration = (10 / diff) * 75 / get_curr_Intelligence( ch );
    delay_to_char( ch, &delay );
    return;
}


void do_select( CHAR_DATA *ch, char *argument )
{
	int i;
	
	if(IS_NPC(ch))
		return;
	
	if( ch->in_room->vnum != ROOM_VNUM_START )
	{
	    send_to_char( "You cannot choose anything now.\r\n", ch );
	    return;
	}
	
    if ( argument[0] == '\0' )
	{
	printf_to_char( ch, "You see a row of idols...\r\n\r\n" );	
	printf_to_char( ch, "*** Please SELECT one of these:\r\n\r\nHuman			the most common of folk, average and diverse\r\nHalf-Giant		a gregarious, strong folk, the kin of giants\r\nHalf-Elf		a mixture of elven and human, of course\r\nHigh Elf		a proud folk, also known as pure elves\r\nWood Elf		a fey folk, also known as wild elves\r\nDark Elf		a ill-reputed folk, also known as deep elves\r\nDwarf			a bearded, often dour, quite strong folk\r\nHalfling		a small and weak, but dextrous and hearty folk\r\nGnome			a small and weak, but smart and agile folk\r\n" );
	
	return;
	}
	
	for ( i = 0; i < MAX_RACE; i++ )
      {
        if ( is_name( argument, race_table[i].name ) )
        {
          ch->race = i;
		  //do_look( ch, race_table[i].name );
		  	printf_to_char(ch, "You reach out and touch the %s idol.\r\nYou feel different.\r\n", race_table[i].name );
			
		ch->pcdata->age = race_table[ch->race].lifespan / 4;
		ch->pcdata->hair =str_dup(race_table[ch->race].hair );
		ch->pcdata->eyes=str_dup(race_table[ch->race].eyes );
		ch->pcdata->skin=str_dup(race_table[ch->race].skin );
		
		printf_to_char(ch, "\r\nYou are %d years old.\r\nYou have %s hair, %s eyes, and %s skin.\r\n", 
		ch->pcdata->age, ch->pcdata->hair, ch->pcdata->eyes, ch->pcdata->skin );
          return;
        }
      }
	
	printf_to_char( ch, "You must SELECT a race.\r\n" );
	return;

}

void do_choose( CHAR_DATA *ch, char *argument )
{
	int i;
	
	if( ch->in_room->vnum != ROOM_VNUM_START )
	{
	    send_to_char( "You cannot choose anything now.\r\n", ch );
	    return;
	}
	
	if(IS_NPC(ch))
		return;
	
    if ( argument[0] == '\0' )
	{
	printf_to_char( ch, "AVAILABLE PROFESSIONS:\r\n" );
	
	for ( i = 0; i < MAX_CLASS; i++ )
      {
		  	printf_to_char(ch, "You could CHOOSE the %s statue.\r\n", class_table[i].name );
      }
	
	return;
	}
	
	for ( i = 0; i < MAX_CLASS; i++ )
      {
        if ( is_name( argument, class_table[i].name ) )
        {
          ch->class = i;
		  	printf_to_char(ch, "You reach out and touch the %s statue.  You feel different.\r\n", class_table[i].name );
          return;
        }
      }
	
	printf_to_char( ch, "You must choose a profession.\r\n" );
	return;

}


void newbie_equip( CHAR_DATA *ch )
{		
		OBJ_DATA *obj;
		ch->gold = 500; //fixme 500 when debts added, add debts.
		ch->move = ch->max_move;
		
		//pouch
		obj = create_object( get_obj_index(3356), 0 );
		obj_to_char( obj, ch );
	    equip_char( ch, obj, EQUIP_BELT );
		SET_BIT( obj->extra_flags, ITEM_GENERATED );
		
		//armor
		obj = create_object( get_obj_index(3353), 0 );
		obj_to_char( obj, ch );
	    equip_char( ch, obj, EQUIP_CHEST );
		SET_BIT( obj->extra_flags, ITEM_GENERATED );
		
		//backpack
		obj = create_object( get_obj_index(3355), 0 );
		obj_to_char( obj, ch );
	    equip_char( ch, obj, EQUIP_BACK );	
		SET_BIT( obj->extra_flags, ITEM_GENERATED );
		

		//falchion
	    obj = create_object( get_obj_index(3352), 0 );
		obj_to_char( obj, ch );
	    obj_to_free_hand( obj, ch );
		SET_BIT( obj->extra_flags, ITEM_GENERATED );
		
		//shield
		obj = create_object( get_obj_index(3354), 0 );
		obj_to_char( obj, ch );
	    obj_to_free_hand( obj, ch );
	
}

void do_begin( CHAR_DATA *ch, char *argument )
{
	if(IS_NPC(ch))
		return;
	
    if( ch->in_room->vnum != ROOM_VNUM_START )
	{
	    send_to_char( "But the adventure has already begun!\r\n", ch );
	    return;
	}
	
    if ( !is_name(argument,"CONFIRM") )
	{
		send_to_char("*** Please make sure to SELECT, ROLL, and SET.\r\n\r\n", ch );
		send_to_char("Then type BEGIN CONFIRM to join the adventure!\r\n", ch );
		return;
	}
	else if( ch->level == 0)
	{	

		
		ch->position = POS_STANDING;

	
	// no race pick? human..
	if( ch->pcdata->age == 0 )
		do_select(ch, "human");

	// fix stats if they didn't roll
	
	if( ch->pcdata->perm_Strength == 0 )
		do_roll(ch, "");	
	
	if( ch->race == RACE_HUMAN || ch->race == RACE_HALF_ELF )
		ch->max_hit = 30;
		
	if( ch->race == RACE_HALFLING || ch->race == RACE_GNOME )
		ch->max_hit -= 10;
		
	if( ch->race == RACE_DWARF || ch->race == RACE_HIGH_MAN )
		ch->max_hit += 15;
		
	if( ch->race == RACE_WOOD_ELF || ch->race == RACE_DARK_ELF || ch->race == RACE_HIGH_ELF)
		ch->max_hit -= 5;
	
	
	ch->max_hit += get_curr_Endurance( ch ) / 10;
			
			ch->hit	= ch->max_hit;
			
			ch->max_mana = get_curr_Potency( ch ) / 30;
			
			if( ch->max_mana < 0 )
				ch->max_mana = 0;
			
			ch->mana	= ch->max_mana;
			
			ch->max_move = get_curr_Potency( ch ) / 10;
				
			if( ch->max_move < 1 ) // less than 1 spirit shouldn't happen
				ch->max_move = 1;
				
			ch->move	= ch->max_move;
			
		ch->pcdata->MTPs = 25 + 
		( ( get_curr_Judgement( ch ) + 
		get_curr_Intelligence( ch ) + 
		get_curr_Wisdom( ch ) + 
		get_curr_Charm( ch )
		+ ( ( get_curr_Potency(ch) + get_curr_Willpower(ch) ) / 2 ) )
		/ 20);
		
		ch->pcdata->PTPs = 25 + 
		( (get_curr_Strength( ch ) + 
		get_curr_Endurance( ch ) + 
		get_curr_Dexterity( ch ) + 
		get_curr_Speed( ch )
		+ ( ( get_curr_Potency(ch) + get_curr_Willpower(ch) ) / 2 ) )
		/ 20);
		ch->practice = 0;
		
		ch->level	= 1;
		
		SET_BIT( ch->act, PLR_CREATED );
		
		newbie_equip( ch );
	
	
	}
	
	send_to_char( "The greatest adventure, is yet to begin...\r\n", ch );
	char_to_room( ch, get_room_index( ROOM_VNUM_GATE ) );
	do_wrap( ch, "" );
	do_look(ch, "");
	
	set_predelay( ch, 150, do_help, "newbie", 0, NULL, NULL, NULL, NULL, TRUE );
}

void do_rename( CHAR_DATA *ch, char *argument )
{
	if(IS_NPC(ch))
		return;
	//fixme
	send_to_char("Rename command not implemented.  Yet...\r\n", ch );
}

/*
		ch->pcdata->perm_Strength = 	number_range(50,90);
		ch->pcdata->perm_Endurance =  	number_range(20,100);
		ch->pcdata->perm_Dexterity =  	number_range(20,50);
		ch->pcdata->perm_Speed =  		number_range(50,90);
		ch->pcdata->perm_Willpower =  	number_range(20,50);
		ch->pcdata->perm_Potency =  	number_range(40,60);
		ch->pcdata->perm_Judgement = 	number_range(50,90);
		ch->pcdata->perm_Intelligence = number_range(40,60);
		ch->pcdata->perm_Wisdom =  		number_range(40,60);
		ch->pcdata->perm_Charm =  		number_range(20,50);
*/

//fixme finish

/*

if( is_name( arg1, "Strength" ) )
	{
		if( is_name( argument, "Strength" ) )
		{
			printf_to_char( ch, "You can't swap it with itself.\r\n" );
			return;
		}
		else if( is_name( argument, "Endurance" ) )
		{
			printf_to_char( ch, "You swap Strength with Endurance.\r\n" );
			a = ch->pcdata->perm_Strength;
			b = ch->pcdata->perm_Endurance;
			ch->pcdata->perm_Strength = b;
			ch->pcdata->perm_Endurance = a;
			return;
		}
		else if( is_name( argument, "Dexterity" ) )
		{
			printf_to_char( ch, "You swap Strength with Dexterity.\r\n" );
			a = ch->pcdata->perm_Strength;
			b = ch->pcdata->perm_Dexterity;
			ch->pcdata->perm_Strength = b;
			ch->pcdata->perm_Dexterity = a;
			return;
		}
		else if( is_name( argument, "Speed" ) )
		{
		}
		else if( is_name( argument, "Willpower" ) )
		{
		}
		else if( is_name( argument, "Willpower" ) )
		{
		}
		else if( is_name( argument, "Judgment" ) )
		{
		}
		else if( is_name( argument, "Intelligence" ) )
		{
		}
		else if( is_name( argument, "Wisdom" ) )
		{
		}
		else if( is_name( argument, "Charm" ) )
		{
		}
		else
		{
		}
	}
	else if( is_name( arg1, "Endurance" ) )
	{
	}
	else if( is_name( arg1, "Dexterity" ) )
	{
	}
	else if( is_name( arg1, "Speed" ) )
	{
	}
	else if( is_name( arg1, "Willpower" ) )
	{
	}
	else if( is_name( arg1, "Willpower" ) )
	{
	}
	else if( is_name( arg1, "Judgment" ) )
	{
	}
	else if( is_name( arg1, "Intelligence" ) )
	{
	}
	else if( is_name( arg1, "Wisdom" ) )
	{
	}
	else if( is_name( arg1, "Charm" ) )
	{
	}
	else
	{
	}
*/


int get_stat( char *arg )
{
    if ( !str_cmp( arg, "Strength" ) || !str_cmp( arg, "str" ) ) return STAT_STRENGTH;
    if ( !str_cmp( arg, "Endurance" ) || !str_cmp( arg, "end"  ) ) return STAT_ENDURANCE;
    if ( !str_cmp( arg, "Dexterity" ) || !str_cmp( arg, "dex" ) ) return STAT_DEXTERITY;
    if ( !str_cmp( arg, "Willpower" ) || !str_cmp( arg, "wil"  ) ) return STAT_WILLPOWER;
    if ( !str_cmp( arg, "Judgment" ) || !str_cmp( arg, "jdg"    ) ) return STAT_JUDGEMENT;
    if ( !str_cmp( arg, "Intelligence" ) || !str_cmp( arg, "int"  ) ) return STAT_INTELLIGENCE;
    if ( !str_cmp( arg, "Charm" ) || !str_cmp( arg, "cha" ) ) return STAT_CHARM;
    if ( !str_cmp( arg, "Wisdom" ) || !str_cmp( arg, "wis"  ) ) return STAT_WISDOM;
    if ( !str_cmp( arg, "Speed" ) || !str_cmp( arg, "spd" ) ) return STAT_SPEED;
    if ( !str_cmp( arg, "Potency" ) || !str_cmp( arg, "pot"  ) ) return STAT_POTENCY;

    return -1;
}


void do_change( CHAR_DATA *ch, char *argument )
{
	char arg1[MAX_INPUT_LENGTH];
	char buf[MSL];
	short *x;
	short *y;
	short a,b;
	argument = one_argument( argument, arg1 );
	
	if(IS_NPC(ch))
		return;
	
	if( ch->in_room->vnum != ROOM_VNUM_START )
	{
	    send_to_char( "It is too late to change your stats.\r\n", ch );
	    return;
	}

/*	
	if( get_stat( arg1 ) == STAT_STRENGTH ) 
	else if get_stat( arg1 ) == STAT_STRENGTH )
*/

	a = get_stat( arg1 );
	b = get_stat( argument );

	if( a == -1 )
	{
	    send_to_char( "Your first stat is bad.\r\n", ch );
	    return;
	}

	if( b == -1 )
	{
	    send_to_char( "Your second stat is bad.\r\n", ch );
	    return;
	}
	
	if( a == b )
	{
		send_to_char( "Your second stat is the same as your first stat.\r\n", ch );
	    return;
	}
	
	//x = get_stat_pointer( ch, arg1 );
	//y = get_stat_pointer( ch, argument );
		
	//swap( x, y );
	
	
		printf_to_char(ch, "NEW STATS:\r\n\r\n" );
		
		    sprintf( buf,
   "    Strength (STR): %d (%d)\r\n   Endurance (END): %d (%d)\r\n   Dexterity (DEX): %d (%d)\r\n       Speed (SPD): %d (%d)\r\n   Willpower (WIL): %d (%d)\r\n     Potency (POT): %d (%d)\r\n    Judgment (JDG): %d (%d)\r\nIntelligence (INT): %d (%d)\r\n      Wisdom (WIS): %d (%d)\r\n       Charm (CHA): %d (%d)\r\n\r\n",
	get_curr_Strength(ch),
	stat_bonus(get_curr_Strength(ch), ch, STAT_STRENGTH),
	get_curr_Endurance(ch),
	stat_bonus(get_curr_Endurance(ch), ch, STAT_ENDURANCE),
	get_curr_Dexterity(ch),
	stat_bonus(get_curr_Dexterity(ch), ch, STAT_DEXTERITY),
	get_curr_Speed(ch),
	stat_bonus(get_curr_Speed(ch), ch, STAT_SPEED),
	get_curr_Willpower(ch),
	stat_bonus(get_curr_Willpower(ch), ch, STAT_WILLPOWER ),
	get_curr_Potency(ch),
	stat_bonus(get_curr_Potency(ch), ch, STAT_POTENCY ),
	get_curr_Judgement(ch),
	stat_bonus(get_curr_Judgement(ch), ch, STAT_JUDGEMENT ),
	get_curr_Intelligence(ch),
	stat_bonus(get_curr_Intelligence(ch), ch, STAT_INTELLIGENCE ),
	get_curr_Wisdom(ch),
	stat_bonus(get_curr_Wisdom(ch), ch, STAT_WISDOM ),
	get_curr_Charm(ch),
	stat_bonus(get_curr_Charm(ch), ch, STAT_CHARM ) );
	
	//printf_to_char(ch, "\r\nUse CHANGE <stat1> <stat2> to swap two statistic numbers.\r\n" );
	
    send_to_char( buf, ch );
}

//add stat switching command...add mangler numbers
void do_roll( CHAR_DATA *ch, char *argument )
{
	char buf[MAX_STRING_LENGTH];
		
		if(IS_NPC(ch))
		return;
	
	if( ch->in_room->vnum != ROOM_VNUM_START && !IS_IMMORTAL(ch) )
	{
	    send_to_char( "You cannot roll right now, shooter.\r\n", ch );
	    return;
	}
	
	printf_to_char(ch, "NEW STATS:\r\n\r\n" );
	
		ch->pcdata->perm_Strength = 	number_range(50,90);
		ch->pcdata->perm_Endurance =  	number_range(20,100);
		ch->pcdata->perm_Dexterity =  	number_range(20,50);
		ch->pcdata->perm_Speed =  		number_range(50,90);
		ch->pcdata->perm_Willpower =  	number_range(20,50);
		ch->pcdata->perm_Potency =  	number_range(40,60);
		ch->pcdata->perm_Judgement = 	number_range(50,90);
		ch->pcdata->perm_Intelligence = number_range(40,60);
		ch->pcdata->perm_Wisdom =  		number_range(40,60);
		ch->pcdata->perm_Charm =  		number_range(20,50);
		
		    sprintf( buf,
   "    Strength (STR): %d (%d)\r\n   Endurance (END): %d (%d)\r\n   Dexterity (DEX): %d (%d)\r\n       Speed (SPD): %d (%d)\r\n   Willpower (WIL): %d (%d)\r\n     Potency (POT): %d (%d)\r\n    Judgment (JDG): %d (%d)\r\nIntelligence (INT): %d (%d)\r\n      Wisdom (WIS): %d (%d)\r\n       Charm (CHA): %d (%d)\r\n\r\n",
	get_curr_Strength(ch),
	stat_bonus(get_curr_Strength(ch), ch, STAT_STRENGTH),
	get_curr_Endurance(ch),
	stat_bonus(get_curr_Endurance(ch), ch, STAT_ENDURANCE),
	get_curr_Dexterity(ch),
	stat_bonus(get_curr_Dexterity(ch), ch, STAT_DEXTERITY),
	get_curr_Speed(ch),
	stat_bonus(get_curr_Speed(ch), ch, STAT_SPEED),
	get_curr_Willpower(ch),
	stat_bonus(get_curr_Willpower(ch), ch, STAT_WILLPOWER ),
	get_curr_Potency(ch),
	stat_bonus(get_curr_Potency(ch), ch, STAT_POTENCY ),
	get_curr_Judgement(ch),
	stat_bonus(get_curr_Judgement(ch), ch, STAT_JUDGEMENT ),
	get_curr_Intelligence(ch),
	stat_bonus(get_curr_Intelligence(ch), ch, STAT_INTELLIGENCE ),
	get_curr_Wisdom(ch),
	stat_bonus(get_curr_Wisdom(ch), ch, STAT_WISDOM ),
	get_curr_Charm(ch),
	stat_bonus(get_curr_Charm(ch), ch, STAT_CHARM ) );
	
	//printf_to_char(ch, "\r\nUse CHANGE <stat1> <stat2> to swap two statistic numbers.\r\n" );
	
    send_to_char( buf, ch );
}


void do_faction( CHAR_DATA *ch, char *argument )
{
	
	
}


// escort mob X to room Y
// find X items
// kill X mobs
// travel to room X
// steal X from mob Y
// open X boxes
// disarm X traps


void do_quest( CHAR_DATA *ch, char *argument )
{
	QUEST_DATA *quest;
	
	/*
	quest->reward = 50;
	
	quest->target = 1;
	*/
	
}


void init_maze()
{
	ROOM_INDEX_DATA *pRoomIndex;
	int i;
	int door;
	int y;
	EXIT_DATA *pexit;
	extern short rev_dir[];

    for( i = 8500; i < 8652; i++ ) //vnums for maze.are
    {
	pRoomIndex = get_room_index( i );
	
	for ( y = 0; y <= number_range(1,7); y++ )
	{
		door = number_range(0,10); // out goes...
		pRoomIndex->exit[door] = NULL;
		pexit = (EXIT_DATA *) alloc_perm( sizeof(*pexit) );
		pexit->to_room = get_room_index( number_range( 8500, 8652 ) ); // back to the entrance
		pexit->description = str_dup( "\0" );
		pexit->keyword = str_dup( "\0" );
		pexit->key = 0;
		pexit->vnum = pRoomIndex->vnum;
		pRoomIndex->exit[rev_dir[door]] = pexit;
	
	}
    }
}

short get_max_field_exp( CHAR_DATA *ch )
{
	if( IS_NPC(ch) )
		return 1000;
	
	return get_curr_Intelligence(ch) + get_curr_Willpower(ch) + 800;
}


#if defined(macintosh)
#include <types.h>
#else
#include <sys/types.h>
#endif
#include <ctype.h>
#include <stdio.h>
#include <string.h>
#include <time.h>
#include <stdarg.h>
#include "merc.h"



AFFECT_DATA *		affect_free;



/*
 * Local functions.
 */
void	affect_modify	args( ( CHAR_DATA *ch, AFFECT_DATA *paf, bool fAdd ) );



/*
 * Retrieve a character's trusted level for permission checking.
 */
int get_trust( CHAR_DATA *ch )
{
    if ( ch->desc != NULL && ch->desc->original != NULL )
	ch = ch->desc->original;

    if ( ch->trust != 0 )
	return ch->trust;

    if ( IS_NPC(ch) && ch->level >= LEVEL_HERO )
	return LEVEL_HERO - 1;
    else
	return ch->level;
}


int get_hours_played( CHAR_DATA *ch )
{
    if ( IS_NPC( ch ) )
	return 0;

    return ( ch->played + (int) (current_time - ch->logon) ) / 3600;
}


/*
 * Retrieve a character's age.
 */
int get_age( CHAR_DATA *ch )
{
    return 17 + ( ch->played + (int) (current_time - ch->logon) ) / 7200;
}



/*
 * Retrieve character's current stats.
 */

int get_curr_Strength( CHAR_DATA *ch )
{
    if ( IS_NPC(ch) )
	return 50;

    return ch->pcdata->perm_Strength + ch->pcdata->mod_Strength;
}

int get_curr_Endurance( CHAR_DATA *ch )
{
    if ( IS_NPC(ch) )
	return 50;

    return ch->pcdata->perm_Endurance + ch->pcdata->mod_Endurance;
}

int get_curr_Dexterity( CHAR_DATA *ch )
{
    if ( IS_NPC(ch) )
	return 50;

    return ch->pcdata->perm_Dexterity + ch->pcdata->mod_Dexterity;
}

int get_curr_Speed( CHAR_DATA *ch )
{
    if ( IS_NPC(ch) )
	return 50;

    return ch->pcdata->perm_Speed + ch->pcdata->mod_Speed;
}

int get_curr_Willpower( CHAR_DATA *ch )
{
    if ( IS_NPC(ch) )
	return 50;

    return ch->pcdata->perm_Willpower + ch->pcdata->mod_Willpower;
}

int get_curr_Potency( CHAR_DATA *ch )
{
    if ( IS_NPC(ch) )
	return 50;

    return ch->pcdata->perm_Potency + ch->pcdata->mod_Potency;
}

int get_curr_Judgement( CHAR_DATA *ch )
{
    if ( IS_NPC(ch) )
	return 50;

    return ch->pcdata->perm_Judgement + ch->pcdata->mod_Judgement;
}

int get_curr_Intelligence( CHAR_DATA *ch )
{
    if ( IS_NPC(ch) )
	return 50;

    return ch->pcdata->perm_Intelligence + ch->pcdata->mod_Intelligence;
}

int get_curr_Wisdom( CHAR_DATA *ch )
{
    if ( IS_NPC(ch) )
	return 50;

    return ch->pcdata->perm_Wisdom + ch->pcdata->mod_Wisdom;
}

int get_curr_Charm( CHAR_DATA *ch )
{

    if ( IS_NPC(ch) )
	return 50;

    return ch->pcdata->perm_Charm + ch->pcdata->mod_Charm;
}


/*
 * Retrieve a character's carry capacity.
 */
int can_carry_n( CHAR_DATA *ch )
{
   /* if ( !IS_NPC(ch) && ch->level >= LEVEL_IMMORTAL )
	return 1000;

    if ( IS_NPC(ch) && IS_SET(ch->act, ACT_PET) )
	return 0; 

    return EQUIP_MAX + 100; */
	
	if(IS_NPC(ch))
		return 100;
	else
		return 500;
}

// Body Weight = Race Base Weight + ((STR stat + CON stat) * Race Weight Factor)
int body_weight( CHAR_DATA *ch )
{
	if(IS_NPC(ch))
		return 200;
	else
	{
		int w = race_table[ch->race].weight_Factor + 
		( ( get_curr_Strength( ch ) + get_curr_Endurance( ch ) ) * race_table[ch->race].weight_Factor / 100 );
		
		return w;
	}
}

/*
 * Retrieve a character's carry capacity.
 */
int can_carry_w( CHAR_DATA *ch )
{
	/*
    if ( !IS_NPC(ch) && ch->level >= LEVEL_IMMORTAL )
	return 1000000;

    if ( IS_NPC(ch) && IS_SET(ch->act, ACT_PET) )
	return 0;
	*/
	
	if(IS_NPC( ch ) )
		return 100;
	else
		return get_curr_Strength(ch); //fixme constitution, strength, and race.
}



/*
 * See if a string is one of the names of an object.
 */
bool is_true_name( const char *str, char *namelist )
{
    char name[MAX_INPUT_LENGTH];

    for ( ; ; )
    {
	namelist = one_argument( namelist, name );
	if ( name[0] == '\0' )
	    return FALSE;
	if ( !str_cmp( str, name ) )
	    return TRUE;
    }
}

/*
 * See if a string is one of the names of an object.
 */
bool is_name ( const char *cstr, char *namelist )
{
    char name[MAX_INPUT_LENGTH], part[MAX_INPUT_LENGTH];
    char *list, *string;
    char *str = str_dup(cstr);


    /* fix crash on NULL namelist */
    if (namelist == NULL || namelist[0] == '\0')
    	return FALSE;

    /* fixed to prevent is_name on "" returning TRUE */
    if (str[0] == '\0')
	return FALSE;

    string = (char *) str;
    /* we need ALL parts of string to match part of namelist */
    for ( ; ; )  /* start parsing string */
    {
    str = (char *) one_argument(str,part);

	if (part[0] == '\0' )
	    return TRUE;

	/* check to see if this is part of namelist */
	list = namelist;
	for ( ; ; )  /* start parsing namelist */
	{
	    list = one_argument(list,name);
	    if (name[0] == '\0')  /* this name was not found */
		return FALSE;

	    if (!str_prefix(string,name))
		return TRUE; /* full pattern match */

	    if (!str_prefix(part,name))
		break;
	}
    }
}



/*
 * Apply or remove an affect to a character.
 */
 
 //removed verbose indicators 12-29-19
 //fixme bring it back? i think i removed because they stack and that was annoying 1-26-22
void affect_modify( CHAR_DATA *ch, AFFECT_DATA *paf, bool fAdd )
{
    int mod;

    mod = paf->modifier;

    if ( fAdd )
    {
	SET_BIT( ch->affected_by, paf->bitvector );
    }
    else
    {
	REMOVE_BIT( ch->affected_by, paf->bitvector );
	mod = 0 - mod;
    }

    switch ( paf->location )
    {
    default:
	bug( "Affect_modify: unknown location %d.", paf->location );
	return;

    case APPLY_NONE:						break;
    case APPLY_STR:           if( !IS_NPC(ch) ) ch->pcdata->mod_Strength	+= mod;	
	
	//mod > 1 ? printf_to_char(ch, "You feel stronger now.\r\n") : printf_to_char(ch, "You feel weaker.\r\n"); break;
    case APPLY_END:           if( !IS_NPC(ch) ) ch->pcdata->mod_Endurance	+= mod; //mod > 1 ? printf_to_char(ch, "You feel tougher now.\r\n") : printf_to_char(ch, "You feel less durable.\r\n"); 	break;
    case APPLY_DEX:           if( !IS_NPC(ch) ) ch->pcdata->mod_Dexterity	+= mod; //mod > 1 ? printf_to_char(ch, "You feel more nimble now.\r\n") : printf_to_char(ch, "You feel clumsier.\r\n");	break;
    case APPLY_SPD:           if( !IS_NPC(ch) )  ch->pcdata->mod_Speed	+= mod; //mod > 1 ? printf_to_char(ch, "You feel quicker now.\r\n") : printf_to_char(ch, "You feel slower.\r\n");	break;
    case APPLY_WIL:           if( !IS_NPC(ch) ) ch->pcdata->mod_Willpower	+= mod; //mod > 1 ? printf_to_char(ch, "You feel more disciplined and resistant to temptation.\r\n") : printf_to_char(ch, "You feel lazier and more vulnerable to temptation.\r\n"); break;
	case APPLY_POT:           if( !IS_NPC(ch) ) ch->pcdata->mod_Potency	+= mod; //mod > 1 ? printf_to_char(ch, "Aah! The power!\r\n") : printf_to_char(ch, "You feel less potent.\r\n"); break;
    case APPLY_JDG:           if( !IS_NPC(ch) ) ch->pcdata->mod_Judgement	+= mod;  //mod > 1 ? printf_to_char(ch, "You feel a bit wiser and contemplative.\r\n") : printf_to_char(ch, "You pause for a moment, trying to collect your wits.\r\n"); break;
    case APPLY_INT:           if( !IS_NPC(ch) ) ch->pcdata->mod_Intelligence += mod; //mod > 1 ? printf_to_char(ch, "It all seems so simple now.\r\n") : printf_to_char(ch, "You suddenly feel like breaking something.\r\n"); break;
    case APPLY_WIS:           if( !IS_NPC(ch) ) ch->pcdata->mod_Wisdom	+= mod;  //mod > 1 ? printf_to_char(ch, "You reflect that the sages had a point, but were a bit sophmoric in their approach.\r\n") : printf_to_char(ch, "But what does it all mean?\r\n"); break;
    case APPLY_CHA:           if( !IS_NPC(ch) ) ch->pcdata->mod_Charm	+= mod;	//mod > 1 ? printf_to_char(ch, "That's right, you've still got it.\r\n") : printf_to_char(ch, "You realize, all at once, that you are fat, old, and ugly.\r\n"); break;
    case APPLY_SEX:           ch->sex			+= mod;	break;
    case APPLY_CLASS:						break;
    case APPLY_LEVEL:						break;
    case APPLY_AGE:						break;
    case APPLY_HEIGHT:						break;
    case APPLY_WEIGHT:						break;
    case APPLY_HIT:          ch->max_hit		+= mod;	
	
	//mod > 1 ? printf_to_char(ch, "It sure does feel good to be alive!\r\n") : printf_to_char(ch, "You are less healthier now.\r\n"); break;
    case APPLY_MANA:           ch->max_mana		+= mod;	
	
	//mod > 1 ? printf_to_char(ch, "You feel more connected to the mana of the world.\r\n") : printf_to_char(ch, "You are less connected to the mana of the world.\r\n"); break;
    case APPLY_MOVE:          ch->max_move		+= mod;	
	
	//mod > 1 ? printf_to_char(ch, "Your connection to the spiritual realm has increased.\r\n") : printf_to_char(ch, "You are now less connected to the spiritual realm.\r\n"); break;
	case APPLY_AS:       ch->as		+= mod;	break;//mod > 1 ? printf_to_char(ch, "You feel more coordinated now.\r\n") : printf_to_char(ch, "You are a little less coordinated now.\r\n");	break;
    case APPLY_DS:       ch->ds		+= mod;	break;//mod > 1 ? printf_to_char(ch, "You feel better protected.\r\n") : printf_to_char(ch, "You feel less protected.\r\n");	break;
	case APPLY_CS:       ch->cs		+= mod;	break;//mod > 1 ? printf_to_char(ch, "You feel a rush of raw power.\r\n") : printf_to_char(ch, "Your power is leaving you.\r\n");break;
    case APPLY_TD:       ch->td		+= mod;	break;//mod > 1 ? printf_to_char(ch, "You feel more able to ward off spells.\r\n") : printf_to_char(ch, "You feel more vulnerable to magic spells.\r\n");break;
    }

    /*
     * Check for weapon wielding.
     * Guard against recursion (for weapons with affects).
     */
	 
	 /*
    if ( ( wield = get_eq_char( ch, WEAR_WIELD ) ) != NULL )
    {
	static int depth;

	if ( depth == 0 )
	{
	    depth++;
	    act( "You drop $p.", ch, wield, NULL, TO_CHAR );
	    act( "$n drops $p.", ch, wield, NULL, TO_ROOM );
	    obj_from_char( wield );
	    obj_to_room( wield, ch->in_room );
	    depth--;
	}
    } */

    return;
}



/*
 * Give an affect to a char.
 */
void affect_to_char( CHAR_DATA *ch, AFFECT_DATA *paf )
{
    AFFECT_DATA *paf_new;

    if ( affect_free == NULL )
    {
	paf_new		= alloc_perm( sizeof(*paf_new) );
    }
    else
    {
	paf_new		= affect_free;
	affect_free	= affect_free->next;
    }

    *paf_new		= *paf;
    paf_new->next	= ch->affected;
    ch->affected	= paf_new;

    affect_modify( ch, paf_new, TRUE );
    return;
}



/*
 * Remove an affect from a char.
 */
void affect_remove( CHAR_DATA *ch, AFFECT_DATA *paf )
{
    if ( ch->affected == NULL )
    {
	bug( "Affect_remove: no affect.", 0 );
	return;
    }

    affect_modify( ch, paf, FALSE );

    if ( paf == ch->affected )
    {
	ch->affected	= paf->next;
    }
    else
    {
	AFFECT_DATA *prev;

	for ( prev = ch->affected; prev != NULL; prev = prev->next )
	{
	    if ( prev->next == paf )
	    {
		prev->next = paf->next;
		break;
	    }
	}

	if ( prev == NULL )
	{
	    bug( "Affect_remove: cannot find paf.", 0 );
	    return;
	}
    }
	
	paf->next   = affect_free;
	affect_free = paf;
    return;
}



/*
 * Strip all affects of a given sn.
 */
void affect_strip( CHAR_DATA *ch, int sn )
{
    AFFECT_DATA *paf;
    AFFECT_DATA *paf_next;

    for ( paf = ch->affected; paf != NULL; paf = paf_next )
    {
	paf_next = paf->next;
	if ( paf->type == sn )
	    affect_remove( ch, paf );
    }

    return;
}



/*
 * Return true if a char is affected by a spell.
 */
bool is_affected( CHAR_DATA *ch, int sn )
{
    AFFECT_DATA *paf;

    for ( paf = ch->affected; paf != NULL; paf = paf->next )
    {
	if ( paf->type == sn )
	    return TRUE;
    }

    return FALSE;
}



/*
 * Add or enhance an affect.
 */
void affect_join( CHAR_DATA *ch, AFFECT_DATA *paf )
{
    AFFECT_DATA *paf_old;

    for ( paf_old = ch->affected; paf_old != NULL; paf_old = paf_old->next )
    {
	if ( paf_old->type == paf->type )
	{
	    paf->duration += paf_old->duration;
	    paf->modifier += paf_old->modifier;
	    affect_remove( ch, paf_old );
	    break;
	}
    }

    affect_to_char( ch, paf );
    return;
}



/*
 * Move a char out of a room.
 */
void char_from_room( CHAR_DATA *ch )
{
    if ( ch->in_room == NULL )
    {
	bug( "Char_from_room: NULL.", 0 );
	return;
    }
	
	if( IS_NPC(ch) )
		--ch->in_room->area->ncritter;
	
    if ( !IS_NPC(ch) )
		--ch->in_room->area->nplayer;

    if ( ch == ch->in_room->people )
    {
	ch->in_room->people = ch->next_in_room;
    }
    else
    {
	CHAR_DATA *prev;

	for ( prev = ch->in_room->people; prev; prev = prev->next_in_room )
	{
	    if ( prev->next_in_room == ch )
	    {
		prev->next_in_room = ch->next_in_room;
		break;
	    }
	}

	if ( prev == NULL )
	    bug( "Char_from_room: ch not found.", 0 );
    }

    ch->in_room      = NULL;
    ch->next_in_room = NULL;
    return;
}



/*
 * Move a char into a room.
 */
void char_to_room( CHAR_DATA *ch, ROOM_INDEX_DATA *pRoomIndex )
{
    if ( pRoomIndex == NULL )
    {
	bug( "Char_to_room: NULL.", 0 );
	return;
    }

    ch->in_room		= pRoomIndex;
    ch->next_in_room	= pRoomIndex->people;
    pRoomIndex->people	= ch;

    if ( !IS_NPC(ch) )
		++ch->in_room->area->nplayer;

	if ( IS_NPC(ch) )
		++ch->in_room->area->ncritter;
	
    return;
}



/*
 * Give an obj to a char.
 */
void obj_to_char( OBJ_DATA *obj, CHAR_DATA *ch )
{
    obj->next_content	 = ch->carrying;
    ch->carrying	 = obj;
    obj->carried_by	 = ch;
    obj->in_room	 = NULL;
    obj->in_obj		 = NULL;
    ch->carry_number	+= get_obj_number( obj );
    ch->carry_weight	+= get_obj_weight( obj );
}


void obj_to_off_hand( OBJ_DATA *obj, CHAR_DATA *ch )
{
	if( get_eq_char( ch, EQUIP_LEFTHAND ) == NULL )
		equip_char( ch, obj, EQUIP_LEFTHAND );
	else if( get_eq_char( ch, EQUIP_RIGHTHAND) == NULL )
		equip_char( ch, obj, EQUIP_RIGHTHAND );
	else
	bug( "obj_to_off_hand: no free hand %d.", 0 );
	
	return;
}

// obj to free hand
void obj_to_free_hand( OBJ_DATA *obj, CHAR_DATA *ch )
{
    if( obj->item_type == ITEM_SHIELD )
	{
    	obj_to_off_hand( obj, ch );
		return;
	}

	if( get_eq_char( ch, EQUIP_RIGHTHAND ) == NULL )
		equip_char( ch, obj, EQUIP_RIGHTHAND );
	
	else if( get_eq_char( ch, EQUIP_LEFTHAND ) == NULL )
		equip_char( ch, obj, EQUIP_LEFTHAND );
	
	else
		bug( "obj_to_free_hand: no free hand %d.", 0 );
	
	return;
}



/*
 * Take an obj from its character.
 */
void obj_from_char( OBJ_DATA *obj )
{
    CHAR_DATA *ch;

    if ( ( ch = obj->carried_by ) == NULL )
    {
	bug( "Obj_from_char: null ch.", 0 );
	return;
    }

    if ( obj->wear_loc != EQUIP_NONE )
	unequip_char( ch, obj );

    if ( ch->carrying == obj )
    {
	ch->carrying = obj->next_content;
    }
    else
    {
	OBJ_DATA *prev;

	for ( prev = ch->carrying; prev != NULL; prev = prev->next_content )
	{
	    if ( prev->next_content == obj )
	    {
		prev->next_content = obj->next_content;
		break;
	    }
	}

	if ( prev == NULL )
	    bug( "Obj_from_char: obj not in list.", 0 );
    }

    obj->carried_by	 = NULL;
    obj->next_content	 = NULL;
    ch->carry_number	-= get_obj_number( obj );
    ch->carry_weight	-= get_obj_weight( obj );
    return;
}





/*
 * Find a piece of eq on a character.
 */
OBJ_DATA *get_eq_char( CHAR_DATA *ch, int iWear )
{
    OBJ_DATA *obj;

    for ( obj = ch->carrying; obj != NULL; obj = obj->next_content )
    {
	if ( obj->wear_loc == iWear )
	    return obj;
    }

    return NULL;
}



/*
 * Equip a char with an obj.
 */
void equip_char( CHAR_DATA *ch, OBJ_DATA *obj, int iWear )
{
    AFFECT_DATA *paf;

	/*
    if ( get_eq_char( ch, iWear ) != NULL )
    {
	bug( "Equip_char: already equipped (%d).", iWear );
	return;
    }
	*/

    obj->wear_loc	 = iWear;

    for ( paf = obj->pIndexData->affected; paf != NULL; paf = paf->next )
	affect_modify( ch, paf, TRUE );
    for ( paf = obj->affected; paf != NULL; paf = paf->next )
	affect_modify( ch, paf, TRUE );

    return;
}



/*
 * Unequip a char with an obj.
 */
void unequip_char( CHAR_DATA *ch, OBJ_DATA *obj )
{
    AFFECT_DATA *paf;

    if ( obj->wear_loc == EQUIP_NONE )
    {
	bug( "Unequip_char: already unequipped.", 0 );
	return;
    }

    obj->wear_loc	 = -1;

    for ( paf = obj->pIndexData->affected; paf != NULL; paf = paf->next )
	affect_modify( ch, paf, FALSE );
    for ( paf = obj->affected; paf != NULL; paf = paf->next )
	affect_modify( ch, paf, FALSE );

    return;
}



/*
 * Count occurrences of an obj in a list.
 */
int count_obj_list( OBJ_INDEX_DATA *pObjIndex, OBJ_DATA *list )
{
    OBJ_DATA *obj;
    int nMatch;

    nMatch = 0;
    for ( obj = list; obj != NULL; obj = obj->next_content )
    {
	if ( obj->pIndexData == pObjIndex )
	    nMatch++;
    }

    return nMatch;
}



/*
 * Move an obj out of a room.
 */
void obj_from_room( OBJ_DATA *obj )
{
    ROOM_INDEX_DATA *in_room;

    if ( ( in_room = obj->in_room ) == NULL )
    {
	bug( "obj_from_room: NULL.", 0 );
	return;
    }

    if ( obj == in_room->contents )
    {
	in_room->contents = obj->next_content;
    }
    else
    {
	OBJ_DATA *prev;

	for ( prev = in_room->contents; prev; prev = prev->next_content )
	{
	    if ( prev->next_content == obj )
	    {
		prev->next_content = obj->next_content;
		break;
	    }
	}

	if ( prev == NULL )
	{
	    bug( "Obj_from_room: obj not found.", 0 );
	    return;
	}
    }

    obj->in_room      = NULL;
    obj->next_content = NULL;
    return;
}



/*
 * Move an obj into a room.
 */
void obj_to_room( OBJ_DATA *obj, ROOM_INDEX_DATA *pRoomIndex )
{
    obj->next_content		= pRoomIndex->contents;
    pRoomIndex->contents	= obj;
    obj->in_room		= pRoomIndex;
    obj->carried_by		= NULL;
    obj->in_obj			= NULL;
    return;
}



/*
 * Move an object into an object.
 */
void obj_to_obj( OBJ_DATA *obj, OBJ_DATA *obj_to )
{
    obj->next_content		= obj_to->contains;
    obj_to->contains		= obj;
    obj->in_obj			= obj_to;
    obj->in_room		= NULL;
    obj->carried_by		= NULL;

    for ( ; obj_to != NULL; obj_to = obj_to->in_obj )
    {
	if ( obj_to->carried_by != NULL )
	{
	    obj_to->carried_by->carry_number += get_obj_number( obj );
	    obj_to->carried_by->carry_weight += get_obj_weight( obj );
	}
    }

    return;
}



/*
 * Move an object out of an object.
 */
void obj_from_obj( OBJ_DATA *obj )
{
    OBJ_DATA *obj_from;

    if ( ( obj_from = obj->in_obj ) == NULL )
    {
	bug( "Obj_from_obj: null obj_from.", 0 );
	return;
    }

    if ( obj == obj_from->contains )
    {
	obj_from->contains = obj->next_content; /// right here***
    }
    else
    {
	OBJ_DATA *prev;

	for ( prev = obj_from->contains; prev; prev = prev->next_content )
	{
	    if ( prev->next_content == obj )
	    {
		prev->next_content = obj->next_content;
		break;
	    }
	}

	if ( prev == NULL )
	{
	    bug( "Obj_from_obj: obj not found.", 0 );
	    return;
	}
    }

    obj->next_content = NULL;
    obj->in_obj       = NULL;

    for ( ; obj_from != NULL; obj_from = obj_from->in_obj )
    {
	if ( obj_from->carried_by != NULL )
	{
	    obj_from->carried_by->carry_number -= get_obj_number( obj );
	    obj_from->carried_by->carry_weight -= get_obj_weight( obj );
	}
    }

    return;
}



/*
 * Extract an obj from the world.
 */
void extract_obj( OBJ_DATA *obj )
{
    OBJ_DATA *obj_content;
    OBJ_DATA *obj_next;
	PREDELAY_DATA *p;
	CHAR_DATA *ch;
	
	if( obj == NULL )
	bug( "Obj is NULL in extract_obj", 0 );
	return;

    if ( obj->in_room != NULL )
		obj_from_room( obj );
    else if ( obj->carried_by != NULL )
		obj_from_char( obj );
    else if ( obj->in_obj != NULL )
		obj_from_obj( obj );

	for ( ch = char_list; ch != NULL; ch = ch->next )
    {
	if ( (p = ch->predelay_info) != NULL )
	{
	    if ( p->obj1 == obj )
		p->obj1 = NULL;
	    if ( p->obj2 == obj )
		p->obj2 = NULL;
	}
    }

    for ( obj_content = obj->contains; obj_content; obj_content = obj_next )
    {
	obj_next = obj_content->next_content;
	extract_obj( obj->contains );
    }

    if ( object_list == obj )
    {
	object_list = obj->next;
    }
    else
    {
	OBJ_DATA *prev;

	for ( prev = object_list; prev != NULL; prev = prev->next )
	{
	    if ( prev->next == obj )
	    {
		prev->next = obj->next;
		break;
	    }
	}

	if ( prev == NULL )
	{
	    bug( "Extract_obj: obj %d not found.", obj->pIndexData->vnum );
	    return;
	}
    }

    {
	AFFECT_DATA *paf;
	AFFECT_DATA *paf_next;

	for ( paf = obj->affected; paf != NULL; paf = paf_next )
	{
	    paf_next    = paf->next;
	    paf->next   = affect_free;
	    affect_free = paf;
	}
    }

    {
	EXTRA_DESCR_DATA *ed;
	EXTRA_DESCR_DATA *ed_next;

	for ( ed = obj->extra_descr; ed != NULL; ed = ed_next )
	{
	    ed_next		= ed->next;
	    free_string( ed->description );
	    free_string( ed->keyword     );
	    extra_descr_free	= ed;
	}
    }

    free_string( obj->name        );
    free_string( obj->description );
    free_string( obj->short_descr );
    --obj->pIndexData->count;
    obj->next	= obj_free;
    obj_free	= obj;
    return;
}



/*
 * Extract a char from the world.
 */
void extract_char( CHAR_DATA *ch, bool fPull )
{
    CHAR_DATA *wch;
    OBJ_DATA *obj;
    OBJ_DATA *obj_next;
	PREDELAY_DATA *p;

    if ( ch->in_room == NULL )
    {
	bug( "Extract_char: NULL.", 0 );
	return;
    }

    if ( fPull )
	die_follower( ch );


    for ( obj = ch->carrying; obj != NULL; obj = obj_next )
    {
	obj_next = obj->next_content;
	extract_obj( obj );
    }

    char_from_room( ch );

    if ( !fPull )
    {
	char_to_room( ch, get_room_index( ROOM_VNUM_ALTAR ) );
	return;
    }

    if ( IS_NPC(ch) )
	--ch->pIndexData->count;

    if ( ch->desc != NULL && ch->desc->original != NULL )
	do_return( ch, "" );

    for ( wch = char_list; wch != NULL; wch = wch->next )
    {
	if ( wch->reply == ch )
	    wch->reply = NULL;
	
	if ( (p = wch->predelay_info) != NULL )
	{
	    if ( p->victim1 == ch )
		p->victim1 = NULL;
	    if ( p->victim2 == ch )
		p->victim2 = NULL;
	}
    }

    if ( ch == char_list )
    {
       char_list = ch->next;
    }
    else
    {
	CHAR_DATA *prev;

	for ( prev = char_list; prev != NULL; prev = prev->next )
	{
	    if ( prev->next == ch )
	    {
		prev->next = ch->next;
		break;
	    }
	}

	if ( prev == NULL )
	{
	    bug( "Extract_char: char not found.", 0 );
	    return;
	}
    }

    if ( ch->desc )
	ch->desc->character = NULL;
    free_char( ch );
    return;
}



/*
 * Find a char in the room.
 */
CHAR_DATA *get_char_room( CHAR_DATA *ch, char *argument )
{
    char arg[MAX_INPUT_LENGTH];
    CHAR_DATA *rch;
    int number;
    int count;

    number = number_argument( argument, arg );
    count  = 0;
    if ( !str_cmp( arg, "self" ) )
	return ch;
    for ( rch = ch->in_room->people; rch != NULL; rch = rch->next_in_room )
    {
	if ( !can_see( ch, rch ) || !is_name( arg, rch->name ) )
	    continue;
	if ( ++count == number )
	    return rch;
    }

    return NULL;
}




/*
 * Find a char in the world.
 */
CHAR_DATA *get_char_world( CHAR_DATA *ch, char *argument )
{
    char arg[MAX_INPUT_LENGTH];
    CHAR_DATA *wch;
    int number;
    int count;

    if ( ( wch = get_char_room( ch, argument ) ) != NULL )
	return wch;

    number = number_argument( argument, arg );
    count  = 0;
    for ( wch = char_list; wch != NULL ; wch = wch->next )
    {
	if ( !can_see( ch, wch ) || !is_name( arg, wch->name ) )
	    continue;
	if ( ++count == number )
	    return wch;
    }

    return NULL;
}



/*
 * Find some object with a given index data.
 * Used by area-reset 'P' command.
 */
OBJ_DATA *get_obj_type( OBJ_INDEX_DATA *pObjIndex )
{
    OBJ_DATA *obj;

    for ( obj = object_list; obj != NULL; obj = obj->next )
    {
	if ( obj->pIndexData == pObjIndex )
	    return obj;
    }

    return NULL;
}


/*
 * Given a string like my.foo, return TRUE and 'foo'
 */

bool smash_my( char *argument, char *arg )
{
    char *pdot;

    for ( pdot = argument; *pdot != '\0'; pdot++ )
    {
	if ( *pdot == '*' )
	{
	    strcpy( arg, pdot+1 );

	    if(argument[0] == 'm' && argument[1] == 'y')
	    return TRUE;
	}
    }

    strcpy( arg, argument );
    return FALSE;
}

/*
 * Find an obj in a list.
 */
OBJ_DATA *get_obj_list( CHAR_DATA *ch, char *argument, OBJ_DATA *list )
{
    char arg[MAX_INPUT_LENGTH];
	char arg2[MSL];
    OBJ_DATA *obj;
    int number;
    int count;
	bool my = FALSE;
	bool parse=FALSE;
	
    number = number_argument( argument, arg );
	
	if( !IS_NPC(ch) && IS_SET(ch->act, PLR_PARSE) )
	{
		parse=TRUE;
		my = smash_my( arg, arg2 );
	}
	
	
	//printf_to_char( ch, "%s - GOL ::  %s :: %d :: %s :: %s :: %s\r\n", parse?"parse on":"noparse",argument, number, my == TRUE? "MY" : "-", arg, parse ? arg2:arg );
	
    count  = 0;
    for ( obj = list; obj != NULL; obj = obj->next_content )
    {
		if( my )
		{
			if( obj->carried_by != NULL && obj->carried_by != ch )
			{
				//printf_to_char( ch, "GOL-Skipping %s (MY but carried by someone else)\r\n", obj->short_descr );
				continue;
			}
			if( obj->carried_by == NULL )
			{
				if( obj->in_obj == NULL )
				{
				//printf_to_char( ch, "GOL-Skipping %s (MY, NULL carried_by, and container is NULL)\r\n", obj->short_descr );
				continue;
				}
			
				if( obj->in_obj->carried_by != ch )
				{
				//printf_to_char( ch, "GOL-Skipping %s (MY, NULL carried_by, and container is carried by someone else\r\n", obj->short_descr );
				continue;
				}
			}
		}
		
	if ( can_see_obj( ch, obj ) )
	{
	if ( is_name( parse ? arg2:arg, obj->name ) )
	{
	    if ( ++count == number )
		{
		return obj;
		}
		else
		{
		//printf_to_char( ch, "GOL-Skipping %s - count doesn't match \r\n", obj->short_descr );
		}
	}
	else
	{
		//printf_to_char( ch, "GOL-Skipping %s - is_name( %s, %s ) is false \r\n", obj->short_descr, parse ? arg2:arg, obj->name );
	}
	}
	else
	{
		//printf_to_char( ch, "GOL-Skipping %s - can't see obj \r\n", obj->short_descr );
	}
	
    }

    return NULL;
}



/*
 * Find an obj in player's inventory.
 */
OBJ_DATA *get_obj_carry( CHAR_DATA *ch, char *argument )
{
    char arg[MAX_INPUT_LENGTH];
	char arg2[MSL];
    OBJ_DATA *obj;
    int number;
    int count;
	bool my = FALSE;
	bool parse=FALSE;
	
    number = number_argument( argument, arg );
	
	if( !IS_NPC(ch) && IS_SET(ch->act, PLR_PARSE) )
	{
		parse=TRUE;
		my = smash_my( arg, arg2 );
	}
	
    count  = 0;
    for ( obj = ch->carrying; obj != NULL; obj = obj->next_content )
    {
		if( my && ( obj->carried_by == NULL || obj->carried_by != ch ) )
			continue;
		
	if ( (obj->wear_loc == EQUIP_NONE || obj->wear_loc == EQUIP_RIGHTHAND || obj->wear_loc == EQUIP_LEFTHAND )
	&&   can_see_obj( ch, obj )
	&&   is_name( parse ? arg2:arg, obj->name ) )
	{
	    if ( ++count == number )
		return obj;
	}
    }

    return NULL;
}



/*
 * Find an obj in player's equipment.
 */
OBJ_DATA *get_obj_wear( CHAR_DATA *ch, char *argument )
{
    char arg[MAX_INPUT_LENGTH];
	char arg2[MSL];
    OBJ_DATA *obj;
    int number;
    int count;
	bool my = FALSE;
	bool parse=FALSE;
	
    number = number_argument( argument, arg );
	
	if( !IS_NPC(ch) && IS_SET(ch->act, PLR_PARSE) )
	{
		parse=TRUE;
		my = smash_my( arg, arg2 );
	}
	
    count  = 0;
    for ( obj = ch->carrying; obj != NULL; obj = obj->next_content )
    {
		if( my && ( obj->carried_by == NULL || obj->carried_by != ch ) )
			continue;
		
	if ( obj->wear_loc != EQUIP_NONE && obj->wear_loc != EQUIP_RIGHTHAND && obj->wear_loc != EQUIP_LEFTHAND
	&&   can_see_obj( ch, obj )
	&&   is_name( parse?arg2:arg, obj->name ) )
	{
	    if ( ++count == number )
		return obj;
	}
    }

    return NULL;
}


OBJ_DATA *get_obj_in_carried_containers( CHAR_DATA *ch, char *argument )
{
    OBJ_DATA *obj;
	OBJ_DATA *obj2;
	
    for ( obj2 = ch->carrying; obj2 != NULL; obj2 = obj2->next_content )
    {
	if( obj2->item_type == ITEM_CONTAINER && !IS_SET(obj2->value[1], CONT_CLOSED) && !IS_SET( obj2->value[1], CONT_LOCKED ) )
	{

	obj = get_obj_list( ch, argument, obj2->contains );
	
	if( !obj )
	continue;
	
	if ( (obj->wear_loc == EQUIP_NONE )
	&&   can_see_obj( ch, obj ) )
	{
	    //if ( ++count == number )
		return obj;
	}
	else
	continue;
	}
    }

    return NULL;
}


OBJ_DATA *get_obj_in_room_containers( CHAR_DATA *ch, char *argument )
{
    OBJ_DATA *obj;
	OBJ_DATA *obj2;
	
    for ( obj2 = ch->in_room->contents; obj2 != NULL; obj2 = obj2->next_content )
    {
	if( (obj2->item_type == ITEM_FURNITURE || obj2->item_type == ITEM_CONTAINER || obj2->item_type == ITEM_MERCHANT_TABLE) && !IS_SET(obj2->value[1], CONT_CLOSED) && !IS_SET( obj2->value[1], CONT_LOCKED ) )
	{

	obj = get_obj_list( ch, argument, obj2->contains );
	
	if( !obj )
	continue;
	
	if ( (obj->wear_loc == EQUIP_NONE )
	&&   can_see_obj( ch, obj ) )
	{
	    //if ( ++count == number )
		//get_obj( ch, obj, obj2 );
		return obj;
	}
	else
	continue;
	}
    }

    return NULL;
}

//returns the container
OBJ_DATA *get_obj_in_carried_containers2( CHAR_DATA *ch, char *argument )
{
    OBJ_DATA *obj;
	OBJ_DATA *obj2;
	
    for ( obj2 = ch->carrying; obj2 != NULL; obj2 = obj2->next_content )
    {
	if( (obj2->item_type == ITEM_FURNITURE || obj2->item_type == ITEM_CONTAINER || obj2->item_type == ITEM_MERCHANT_TABLE) && !IS_SET(obj2->value[1], CONT_CLOSED) && !IS_SET( obj2->value[1], CONT_LOCKED ) )
	{

	obj = get_obj_list( ch, argument, obj2->contains );
	
	if( !obj )
	continue;
	
	if ( (obj->wear_loc == EQUIP_NONE )
	&&   can_see_obj( ch, obj ) )
	{
	    //if ( ++count == number )
		return obj2;
	}
	else
	continue;
	}
    }

    return NULL;
}


//returns the container
OBJ_DATA *get_obj_in_room_containers2( CHAR_DATA *ch, char *argument )
{
    OBJ_DATA *obj;
	OBJ_DATA *obj2;
	
    for ( obj2 = ch->in_room->contents; obj2 != NULL; obj2 = obj2->next_content )
    {
	if( (obj2->item_type == ITEM_FURNITURE || obj2->item_type == ITEM_CONTAINER || obj2->item_type == ITEM_MERCHANT_TABLE ) && !IS_SET(obj2->value[1], CONT_CLOSED) && !IS_SET( obj2->value[1], CONT_LOCKED ) )
	{

	obj = get_obj_list( ch, argument, obj2->contains );
	
	if( !obj )
	continue;
	
	if ( (obj->wear_loc == EQUIP_NONE )
	&&   can_see_obj( ch, obj ) )
	{
	    //if ( ++count == number )
		//get_obj( ch, obj, obj2 );
		return obj2;
	}
	else
	continue;
	}
    }

    return NULL;
}


/*
 * Find an obj in the room or in inventory.
 */
OBJ_DATA *get_obj_here( CHAR_DATA *ch, char *argument )
{
    OBJ_DATA *obj;

    obj = get_obj_list( ch, argument, ch->in_room->contents );
    if ( obj != NULL )
	return obj;

    if ( ( obj = get_obj_in_room_containers( ch, argument ) ) != NULL )
	return obj;

    if ( ( obj = get_obj_in_carried_containers( ch, argument ) ) != NULL )
	return obj;

    if ( ( obj = get_obj_carry( ch, argument ) ) != NULL )
	return obj;

    if ( ( obj = get_obj_wear( ch, argument ) ) != NULL )
	return obj;



    return NULL;
}

OBJ_DATA *get_obj_close( CHAR_DATA *ch, char *argument )
{
    OBJ_DATA *obj;

    if ( ( obj = get_obj_carry( ch, argument ) ) != NULL )
	return obj;

    if ( ( obj = get_obj_wear( ch, argument ) ) != NULL )
	return obj;

    return NULL;
}



/*
 * Find an obj in the world.
 */
OBJ_DATA *get_obj_world( CHAR_DATA *ch, char *argument )
{
    char arg[MAX_INPUT_LENGTH];
    OBJ_DATA *obj;
    int number;
    int count;

    if ( ( obj = get_obj_here( ch, argument ) ) != NULL )
	return obj;

    number = number_argument( argument, arg );
    count  = 0;
    for ( obj = object_list; obj != NULL; obj = obj->next )
    {
	if ( can_see_obj( ch, obj ) && is_name( arg, obj->name ) )
	{
	    if ( ++count == number )
		return obj;
	}
    }

    return NULL;
}



/*
 * Create a 'money' obj.
 */
OBJ_DATA *create_money( int amount )
{
    OBJ_DATA *obj;

    if ( amount <= 0 )
    {
	bug( "Create_money: zero or negative money %d.", amount );
	amount = 1;
    }

    if ( amount == 1 )
    {
	obj = create_object( get_obj_index( OBJ_VNUM_MONEY_ONE ), 0 );
    }
    else
    {
	obj = create_object( get_obj_index( OBJ_VNUM_MONEY_SOME ), 0 );
	obj->value[0]		= amount;
    }

    return obj;
}



/*
 * Return # of objects which an object counts as.
 * Thanks to Tony Chamberlain for the correct recursive code here.
 */
int get_obj_number( OBJ_DATA *obj )
{
    int number;

    if ( obj->item_type == ITEM_CONTAINER )
	number = 0;
    else
	number = 1;

    for ( obj = obj->contains; obj != NULL; obj = obj->next_content )
	number += get_obj_number( obj );

    return number;
}



/*
 * Return weight of an object, including weight of contents.
 */
int get_obj_weight( OBJ_DATA *obj )
{
    int weight;

    weight = obj->weight;
    for ( obj = obj->contains; obj != NULL; obj = obj->next_content )
	weight += get_obj_weight( obj );

    return weight;
}



/*
 * True if room is dark.
 */
 //unused
bool room_is_dark( ROOM_INDEX_DATA *pRoomIndex )
{
	return FALSE;

    if ( pRoomIndex->light > 0 )
	return FALSE;

    //if ( IS_SET(pRoomIndex->room_flags, ROOM_DARK) )
	//return TRUE;

    if ( pRoomIndex->sector_type == SECT_INSIDE
    ||   pRoomIndex->sector_type == SECT_CITY )
	return FALSE;

    if ( weather_info.sunlight == SUN_DARK )
	return TRUE;

    return FALSE;
}



/*
 * True if room is private.
 */
bool room_is_private( ROOM_INDEX_DATA *pRoomIndex )
{
    CHAR_DATA *rch;
    int count;

    count = 0;
    for ( rch = pRoomIndex->people; rch != NULL; rch = rch->next_in_room )
	count++;

    if ( IS_SET(pRoomIndex->room_flags, ROOM_PRIVATE)  && count >= 2 )
	return TRUE;

    if ( IS_SET(pRoomIndex->room_flags, ROOM_SOLITARY) && count >= 1 )
	return TRUE;

    return FALSE;
}



/*
 * True if char can see victim.
 */
bool can_see( CHAR_DATA *ch, CHAR_DATA *victim )
{
    if ( ch == victim )
	return TRUE;

    if ( !IS_NPC(victim)
    &&   IS_SET(victim->act, PLR_WIZINVIS)
    &&   get_trust( ch ) < get_trust( victim ) )
	return FALSE;

    if ( !IS_NPC(ch) && IS_SET(ch->act, PLR_HOLYLIGHT) )
	return TRUE;

    if ( IS_AFFECTED(ch, AFF_BLIND) )
	return FALSE;

// there aren't such spells and there won't be

/*
    if ( room_is_dark( ch->in_room ) && !IS_AFFECTED(ch, AFF_INFRARED) )
	return FALSE;

    if ( IS_AFFECTED(victim, AFF_INVISIBLE)
    &&   !IS_AFFECTED(ch, AFF_DETECT_INVIS) )
	return FALSE;
*/

	//fixme invis
    if ( IS_AFFECTED(victim, AFF_INVISIBLE) )
	return FALSE;

    if ( IS_AFFECTED(victim, AFF_HIDE)
    &&   !IS_AFFECTED(ch, AFF_DETECT_HIDDEN) )
	return FALSE;

    return TRUE;
}



/*
 * True if char can see obj.
 */
bool can_see_obj( CHAR_DATA *ch, OBJ_DATA *obj )
{
    if ( !IS_NPC(ch) && IS_SET(ch->act, PLR_HOLYLIGHT) )
		return TRUE;

//fixme? when blindness and cure blindness
/*
    if ( obj->item_type == ITEM_POTION ) 
	return TRUE;
*/

    if ( IS_AFFECTED( ch, AFF_BLIND ) )
		return FALSE;

    if ( obj->item_type == ITEM_LIGHT && obj->value[2] != 0 )
		return TRUE;

    //if ( room_is_dark( ch->in_room ) && !IS_AFFECTED(ch, AFF_INFRARED) )
	//return FALSE;

    if ( IS_SET(obj->extra_flags, ITEM_INVIS) && !IS_AFFECTED(ch, AFF_DETECT_INVIS) )
		return FALSE;
	
	if ( IS_SET(obj->extra_flags, ITEM_HIDDEN) && !IS_AFFECTED(ch, AFF_DETECT_HIDDEN) )
		return FALSE;

    return TRUE;
}



/*
 * True if char can drop obj.
 */
bool can_drop_obj( CHAR_DATA *ch, OBJ_DATA *obj )
{
    if ( !IS_SET(obj->extra_flags, ITEM_NODROP) )
	return TRUE;

/*
    if ( !IS_NPC(ch) && ch->level >= LEVEL_IMMORTAL )
	return TRUE;
*/

    return FALSE;
}



/*
 * Return ascii name of an item type.
 */
char *item_type_name( OBJ_DATA *obj )
{
    switch ( obj->item_type )
    {
    case ITEM_LIGHT:		return "light";
    case ITEM_SCROLL:		return "scroll";
    case ITEM_WAND:		return "wand"; //wavable
    case ITEM_STAFF:		return "staff";
    case ITEM_WEAPON:		return "weapon";
	case ITEM_GEM:		return "gem";
    case ITEM_TREASURE:		return "treasure";
    case ITEM_ARMOR:		return "armor";
    case ITEM_POTION:		return "potion";
    case ITEM_FURNITURE:	return "furniture";
    case ITEM_TRASH:		return "trash";
    case ITEM_CONTAINER:	return "container";
    case ITEM_DRINK_CON:	return "drink container";
    case ITEM_KEY:		return "key";
    case ITEM_FOOD:		return "food";
    case ITEM_MONEY:		return "money";
    case ITEM_BOAT:		return "boat";
    case ITEM_CORPSE_NPC:	return "npc corpse";
    case ITEM_CORPSE_PC:	return "pc corpse";
    case ITEM_FOUNTAIN:		return "fountain";
    case ITEM_PILL:		return "pill";
	case ITEM_PORTAL:	return "portal";
	case ITEM_BUILDING:   return "building";
	case ITEM_DOOR	:	return "door";
	case ITEM_CLIMB	:		return "climb";
	case ITEM_SHIELD:	return "shield";
	case ITEM_SWIM	:		return "swim";
	case ITEM_RUB:		return "rubbable magic item";
	case ITEM_LOCKPICK: return "lockpick";
	case ITEM_NEST: return "creature nest";
	case ITEM_MERCHANT_TABLE: return "merchant table";
	case ITEM_SKIN: return "skin";
    }

    bug( "Item_type_name: unknown type %d.", obj->item_type );
    return "(unknown)";
}



/*
 * Return ascii name of an affect location.
 */
char *affect_loc_name( int location )
{
    switch ( location )
    {
    case APPLY_NONE:	return "none";
    case APPLY_NONE1:	return "none1";
    case APPLY_NONE2:	return "none2";
    case APPLY_NONE3:	return "none3";
    case APPLY_NONE4:	return "none4";
    case APPLY_NONE5:	return "none5";
    case APPLY_SEX:		return "sex";
    case APPLY_CLASS:	return "class";
    case APPLY_LEVEL:	return "level";
    case APPLY_AGE:		return "age";
    case APPLY_MANA:	return "Mana";
    case APPLY_HIT:		return "Health";
    case APPLY_MOVE:	return "Spirit";
    case APPLY_AS:		return "Attack";
    case APPLY_DS:		return "Defense";
    case APPLY_CS:		return "Spell Power";
    case APPLY_TD:		return "Magic Defense";
	case APPLY_STR:		return "Strength";
    case APPLY_END:		return "Endurance";
    case APPLY_SPD:		return "Speed";
    case APPLY_DEX:		return "Dexterity";
    case APPLY_WIL:		return "Willpower";
	case APPLY_POT:		return "Potency";
    case APPLY_JDG:		return "Judgment";
    case APPLY_INT:		return "Intelligence";
    case APPLY_WIS:		return "Wisdom";
    case APPLY_CHA:		return "Charm";
    }

    bug( "Affect_location_name: unknown location %d.", location );
    return "(unknown)";
}

char *select_a_an( char *str )
{
    if ( str[0] == 'a'
    || str[0] == 'e'
    || str[0] == 'i'
    || str[0] == 'o'
    || str[0] == 'u' )
	return "an";

    /*
     * need a heuristic for weird cases,
     * especially starting with the letter "h"
     */

    return "a";
}

char *smash_article( char *text )
{
    char *arg;
    char buf[MAX_STRING_LENGTH];
    static char buf2[MAX_STRING_LENGTH];

    one_argument( text, buf );

    if ( !str_cmp( buf, "the" ) ||
         !str_cmp( buf, "an"  ) ||
         !str_cmp( buf, "a"   ) ||
         !str_cmp( buf, "some" ) )
    {
        arg = one_argument( text, buf );
        sprintf( buf2, "%s", arg );
    }
    else strcpy( buf2, text );

    return buf2;
}

/*
 * Return ascii name of an affect bit vector.
 */
char *affect_bit_name( int vector )
{
    static char buf[512];

    buf[0] = '\0';
    if ( vector & AFF_BLIND         ) strcat( buf, " blind"         );
    if ( vector & AFF_INVISIBLE     ) strcat( buf, " invisible"     );
    if ( vector & AFF_DETECT_EVIL   ) strcat( buf, " detect_evil"   );
    if ( vector & AFF_DETECT_INVIS  ) strcat( buf, " detect_invis"  );
    if ( vector & AFF_DETECT_MAGIC  ) strcat( buf, " detect_magic"  );
    if ( vector & AFF_DETECT_HIDDEN ) strcat( buf, " detect_hidden" );
    if ( vector & AFF_DEAD          ) strcat( buf, " dead"          );
    if ( vector & AFF_SANCTUARY     ) strcat( buf, " sanctuary"     );
    if ( vector & AFF_SKINNED   	) strcat( buf, " skinned"   );			//AFF_SKINNED <-- AFF_FAERIE_FIRE
    if ( vector & AFF_MARKED      	) strcat( buf, " marked"      );		//AFF_MARKED <-- AFF_INFRARED
    if ( vector & AFF_CURSE         ) strcat( buf, " curse"         );
    if ( vector & AFF_UNDEAD        ) strcat( buf, " undead"       );
    if ( vector & AFF_POISON        ) strcat( buf, " poison"        );
    if ( vector & AFF_PROTECT       ) strcat( buf, " protect"       );
    if ( vector & AFF_NONCORP       ) strcat( buf, " noncorp"     );
    if ( vector & AFF_SLEEP         ) strcat( buf, " sleep"         );
    if ( vector & AFF_SNEAK         ) strcat( buf, " sneak"         );
    if ( vector & AFF_HIDE          ) strcat( buf, " hide"          );
    if ( vector & AFF_CHARM         ) strcat( buf, " charm"         );
    if ( vector & AFF_FLYING        ) strcat( buf, " flying"        );
    if ( vector & AFF_PASS_DOOR     ) strcat( buf, " pass_door"     );
	if ( vector & AFF_BLESS     	) strcat( buf, " bless"     );
    return ( buf[0] != '\0' ) ? buf+1 : "none";
}



/*
 * Return ascii name of extra flags vector.
 */
char *extra_bit_name( int extra_flags )
{
    static char buf[512];

    buf[0] = '\0';
	if ( extra_flags & ITEM_VT_ROOM    	 ) strcat( buf, " vt-room"    );
	if ( extra_flags & ITEM_VT_INVENTORY    	 ) strcat( buf, " vt-inv"    );
    if ( extra_flags & ITEM_DARK         ) strcat( buf, " dark"         );
    if ( extra_flags & ITEM_LOCK         ) strcat( buf, " lock"         );
    if ( extra_flags & ITEM_EVIL         ) strcat( buf, " evil"         );
    if ( extra_flags & ITEM_INVIS        ) strcat( buf, " invis"        );
    if ( extra_flags & ITEM_MAGIC        ) strcat( buf, " magic"        );
    if ( extra_flags & ITEM_NODROP       ) strcat( buf, " nodrop"       );
    if ( extra_flags & ITEM_BLESS        ) strcat( buf, " bless"        );
    if ( extra_flags & ITEM_GENERATED    ) strcat( buf, " generated"    );
    if ( extra_flags & ITEM_ANTI_EVIL    ) strcat( buf, " anti-evil"    );
    if ( extra_flags & ITEM_ANTI_NEUTRAL ) strcat( buf, " anti-neutral" );
    if ( extra_flags & ITEM_NOREMOVE     ) strcat( buf, " noremove"     );
    if ( extra_flags & ITEM_INVENTORY    ) strcat( buf, " inventory"    );
    return ( buf[0] != '\0' ) ? buf+1 : "none";
}

/* source: EOD, by John Booth <???> */
void printf_to_char (CHAR_DATA *ch, char *fmt, ...)
{
	char buf [MAX_STRING_LENGTH];
	va_list args;
	va_start (args, fmt);
	vsnprintf (buf, MAX_STRING_LENGTH, fmt, args);
	va_end (args);

	send_to_char (buf, ch);
}


void set_predelay( CHAR_DATA *ch, int delay, DO_FUN *fnptr, char *argument,
    int number, CHAR_DATA *victim1, CHAR_DATA *victim2, OBJ_DATA *obj1,
    OBJ_DATA *obj2, bool noInterrupt )
{
    PREDELAY_DATA *p;
	
    p = new_predelay( );

    p->fnptr = fnptr;
    p->argument[0] = '\0';
    strcpy( p->argument, argument );
    p->number = number;
    p->victim1 = victim1;
    p->victim2 = victim2;
    p->obj1 = obj1;
    p->obj2 = obj2;
	p->noInterrupt = noInterrupt;
	
    if ( ch->predelay_info != NULL )
	free_predelay( ch->predelay_info );

    ch->predelay_info = p;
    ch->predelay_time = delay;
	
}


#include <ctype.h>
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <time.h>
#include "merc.h"



bool	check_social	args( ( CHAR_DATA *ch, char *command,
			    char *argument ) );
				
bool	check_verb_traps	args( ( CHAR_DATA *ch, char *command, char *argument ) );



/*
 * Log-all switch.
 */
bool				fLogAll		= FALSE;



/*
 * Command table.
 */
const	struct	cmd_type	cmd_table	[] =
{
    /*
     * Common movement commands.
     */
    { "north",			do_north,		POS_STANDING,	 0,  LOG_NORMAL, CMD_BREAKS_HIDE|CMD_BREAKS_PREDELAY },
    { "east",			do_east,		POS_STANDING,	 0,  LOG_NORMAL, CMD_BREAKS_HIDE|CMD_BREAKS_PREDELAY },
    { "south",			do_south,		POS_STANDING,	 0,  LOG_NORMAL, CMD_BREAKS_HIDE|CMD_BREAKS_PREDELAY },
    { "west",			do_west,		POS_STANDING,	 0,  LOG_NORMAL, CMD_BREAKS_HIDE|CMD_BREAKS_PREDELAY },
    { "up",				do_up,			POS_STANDING,	 0,  LOG_NORMAL, CMD_BREAKS_HIDE|CMD_BREAKS_PREDELAY },
    { "down",			do_down,		POS_STANDING,	 0,  LOG_NORMAL, CMD_BREAKS_HIDE|CMD_BREAKS_PREDELAY },
	{ "ne",				do_northeast,	POS_STANDING,	 0,  LOG_NORMAL, CMD_BREAKS_HIDE|CMD_BREAKS_PREDELAY },
    { "nw",				do_northwest,	POS_STANDING,	 0,  LOG_NORMAL, CMD_BREAKS_HIDE|CMD_BREAKS_PREDELAY },
    { "se",				do_southeast,	POS_STANDING,	 0,  LOG_NORMAL, CMD_BREAKS_HIDE|CMD_BREAKS_PREDELAY },
    { "sw",				do_southwest,	POS_STANDING,	 0,  LOG_NORMAL, CMD_BREAKS_HIDE|CMD_BREAKS_PREDELAY },
	{ "northeast",		do_northeast,	POS_STANDING,	 0,  LOG_NORMAL, CMD_BREAKS_HIDE|CMD_BREAKS_PREDELAY },
    { "northwest",		do_northwest,	POS_STANDING,	 0,  LOG_NORMAL, CMD_BREAKS_HIDE|CMD_BREAKS_PREDELAY },
    { "southeast",		do_southeast,	POS_STANDING,	 0,  LOG_NORMAL, CMD_BREAKS_HIDE|CMD_BREAKS_PREDELAY },
    { "southwest",		do_southwest,	POS_STANDING,	 0,  LOG_NORMAL, CMD_BREAKS_HIDE|CMD_BREAKS_PREDELAY },
    { "out",			do_out,			POS_STANDING,	 0,  LOG_NORMAL, CMD_BREAKS_HIDE|CMD_BREAKS_PREDELAY },
    { "go",			    do_enter,		POS_STANDING,	 0,  LOG_NORMAL, CMD_BREAKS_HIDE|CMD_BREAKS_PREDELAY },
	
    /*
     * Common other commands.
     * Placed here so one and two letter abbreviations work.
     */
	{ "attack",		do_kill,	POS_PRONE,	 0,  LOG_NORMAL, CMD_BREAKS_HIDE|CMD_BREAKS_PREDELAY },
	{ "ambush",		do_ambush,	POS_PRONE,	 0,  LOG_NORMAL, CMD_BREAKS_PREDELAY	}, // breaks hide by itself
    { "buy",		do_buy,		POS_PRONE,	 0,  LOG_NORMAL, CMD_BREAKS_HIDE|CMD_BREAKS_PREDELAY },
    { "cast",		do_cast,	POS_PRONE,	 0,  LOG_NORMAL, CMD_BREAKS_HIDE|CMD_BREAKS_PREDELAY },
	{ "climb",		do_enter,		POS_STANDING,	 0,  LOG_NORMAL, CMD_BREAKS_HIDE|CMD_BREAKS_PREDELAY },
    { "prepare",	do_prepare,	POS_PRONE,	 0,  LOG_NORMAL, CMD_BREAKS_HIDE|CMD_BREAKS_PREDELAY },
    { "get",		do_get,		POS_PRONE,	 0,  LOG_NORMAL, CMD_BREAKS_HIDE|CMD_BREAKS_PREDELAY },
	{ "health",		do_health,	POS_PRONE,	 0,  LOG_NORMAL, BYPASS_STUN|BYPASS_RT	},
    { "inventory",	do_equipment,	POS_PRONE,	 0,  LOG_NORMAL, BYPASS_STUN|BYPASS_RT	},
    { "kill",		do_kill,	POS_PRONE,	 0,  LOG_NORMAL, CMD_BREAKS_HIDE|CMD_BREAKS_PREDELAY },
	{ "kneel",		do_kneel,	POS_PRONE,	 0,  LOG_NORMAL, CMD_BREAKS_HIDE|CMD_BREAKS_PREDELAY },
	{ "look",		do_look,	POS_PRONE,	     0,  LOG_NORMAL, BYPASS_STUN|BYPASS_RT|BYPASS_DEAD	},
	{ "read",		do_read,	POS_PRONE,	     0,  LOG_NORMAL, BYPASS_STUN|BYPASS_RT|BYPASS_DEAD	},
	{ "lie",		do_lie,		POS_PRONE,	 0,  LOG_NORMAL, CMD_BREAKS_HIDE|CMD_BREAKS_PREDELAY },
    { "order",		do_order2,	POS_PRONE,	 0,  LOG_ALWAYS, CMD_BREAKS_HIDE|CMD_BREAKS_PREDELAY },
	{ "rest",		do_sit,		POS_PRONE,	 0,  LOG_NORMAL, CMD_BREAKS_HIDE|CMD_BREAKS_PREDELAY },
	{ "sit",		do_sit,		POS_PRONE,	 0,  LOG_NORMAL, CMD_BREAKS_HIDE|CMD_BREAKS_PREDELAY },
    { "stand",		do_stand,	POS_PRONE,	 0,  LOG_NORMAL, CMD_BREAKS_HIDE|CMD_BREAKS_PREDELAY },
	{ "stance",		do_stance,	POS_PRONE,	 0,  LOG_NORMAL, CMD_BREAKS_HIDE|CMD_BREAKS_PREDELAY },
	{ "tell",		do_order,	POS_PRONE,	 0,  LOG_NORMAL, CMD_BREAKS_HIDE|CMD_BREAKS_PREDELAY 	},
    { "whisper",	do_tell,	POS_PRONE,	 0,  LOG_NORMAL, CMD_BREAKS_PREDELAY 	},
    { "wield",		do_wear,	POS_PRONE,	 0,  LOG_NORMAL, CMD_BREAKS_HIDE|CMD_BREAKS_PREDELAY },
	
	
    /*
     * Informational commands.
     */
	{ "analyze",	do_examine,	POS_PRONE,	 0,  LOG_NORMAL, BYPASS_STUN|BYPASS_RT	},
    { "areas",		do_areas,	POS_PRONE,	 0,  LOG_NORMAL, BYPASS_STUN|BYPASS_RT	},
    { "bug",		do_bug,		POS_PRONE,	 0,  LOG_NORMAL, BYPASS_STUN|BYPASS_RT|CMD_DONT_PARSE	},
	{ "change",		do_change,	POS_PRONE,	 0,  LOG_NORMAL, CMD_BREAKS_HIDE|CMD_BREAKS_PREDELAY	},
    { "credits",	do_credits,	POS_PRONE,	 0,  LOG_NORMAL, BYPASS_STUN|BYPASS_RT	},
	{ "encumbrance",		do_encumbrance,	POS_PRONE,	 0,  LOG_NORMAL, CMD_BREAKS_HIDE|CMD_BREAKS_PREDELAY	},
    { "equipment",	do_equipment,	POS_PRONE,	 0,  LOG_NORMAL, BYPASS_STUN|BYPASS_RT	},
    { "examine",	do_examine,	POS_PRONE,	 0,  LOG_NORMAL, BYPASS_STUN|BYPASS_RT	},
	{ "experience",	do_experience,	POS_PRONE,	 0,  LOG_NORMAL, BYPASS_STUN|BYPASS_RT	},
	{ "faction",	do_faction,	POS_PRONE,	 0,  LOG_NORMAL, BYPASS_STUN|BYPASS_RT	},
	{ "hands",		do_glance,	POS_PRONE,	 0,  LOG_NORMAL, BYPASS_STUN|BYPASS_RT|CMD_DONT_PARSE	},
    { "help",		do_help,	POS_PRONE,	 0,  LOG_NORMAL, BYPASS_STUN|BYPASS_RT	},
    { "glance",		do_glance,	POS_PRONE,	 0,  LOG_NORMAL, BYPASS_STUN|BYPASS_RT|CMD_DONT_PARSE	},
    { "info",		do_score,	POS_PRONE,	 0,  LOG_NORMAL, BYPASS_STUN|BYPASS_RT	},
	{ "inspect",	do_examine,	POS_PRONE,	 0,  LOG_NORMAL, BYPASS_STUN|BYPASS_RT	},
	{ "location",	do_areas,	POS_PRONE,	 0,  LOG_NORMAL, BYPASS_STUN|BYPASS_RT	},
	{ "mana",		do_mana,	POS_PRONE,	 0,  LOG_NORMAL, BYPASS_STUN|BYPASS_RT	},
	{ "peer",		do_peer,	POS_PRONE,	 0,  LOG_NORMAL, CMD_BREAKS_HIDE|CMD_BREAKS_PREDELAY	},
	{ "power",		do_mana,	POS_PRONE,	 0,  LOG_NORMAL, BYPASS_STUN|BYPASS_RT	},
    { "report",		do_bug,		POS_PRONE,	 0,  LOG_NORMAL, BYPASS_DEAD|BYPASS_STUN|BYPASS_RT	},
	{ "score",		do_score2,	POS_PRONE,	 0,  LOG_NORMAL, BYPASS_STUN|BYPASS_RT	},
	{ "spells",		do_spells,	POS_PRONE,	 0,  LOG_NORMAL, BYPASS_STUN|BYPASS_RT	},
    { "time",		do_time,	POS_PRONE,	 0,  LOG_NORMAL, BYPASS_DEAD|BYPASS_STUN|BYPASS_RT	},
    { "typo",		do_bug,		POS_PRONE,	 0,  LOG_NORMAL, BYPASS_STUN|BYPASS_RT|CMD_DONT_PARSE	},
	{ "wealth",		do_wealth,	POS_PRONE,	 0,  LOG_NORMAL, CMD_BREAKS_HIDE|CMD_BREAKS_PREDELAY	},
	{ "weight",		do_weigh,	POS_PRONE,	 0,  LOG_NORMAL, CMD_BREAKS_HIDE|CMD_BREAKS_PREDELAY	},
    { "weather",	do_weather,	POS_PRONE,	 0,  LOG_NORMAL, BYPASS_DEAD|BYPASS_STUN|BYPASS_RT	},
    { "who",		do_who,		POS_PRONE,	 0,  LOG_NORMAL, BYPASS_STUN|BYPASS_RT	},

    /*
     * Configuration commands.
     */

    { "channels",	do_channels,	POS_PRONE,	 MAX_LEVEL-3,  LOG_NORMAL, BYPASS_STUN|BYPASS_RT|CMD_DONT_PARSE	},
    { "password",	do_password,	POS_PRONE,	 0,  LOG_NEVER, BYPASS_STUN|BYPASS_RT|CMD_DONT_PARSE	},
    { "set",		do_config,	POS_PRONE,	 0,  LOG_NORMAL, BYPASS_STUN|BYPASS_RT|CMD_DONT_PARSE	},
	{ "wrap",		do_wrap,	POS_PRONE,	 0,  LOG_NORMAL, BYPASS_STUN|BYPASS_RT|CMD_DONT_PARSE	},
	
    /*
     * Communication commands.
     */
	 
	{ "act",		do_emote,	POS_PRONE,	 0,  LOG_NORMAL, CMD_DONT_PARSE	},
    { "chat",		do_chat,	POS_PRONE,	 0,  LOG_NORMAL, BYPASS_RT|CMD_DONT_PARSE	},
    { "say",		do_say,		POS_PRONE,	 0,  LOG_NORMAL, BYPASS_RT|CMD_DONT_PARSE	},
	{ "\"",		do_say,		POS_PRONE,	 0,  LOG_NORMAL, BYPASS_RT|CMD_DONT_PARSE	},
    { "'",		do_say,		POS_PRONE,	 0,  LOG_NORMAL, BYPASS_RT|CMD_DONT_PARSE	},
    { "shout",		do_shout,	POS_PRONE,	 0,  LOG_NORMAL, BYPASS_RT|CMD_DONT_PARSE|CMD_BREAKS_HIDE	},
    { "yell",		do_shout,	POS_PRONE,	 0,  LOG_NORMAL, BYPASS_RT|CMD_DONT_PARSE|CMD_BREAKS_HIDE	},

    /*
     * Object manipulation commands.
     */
	{ "appraise",		do_value,	POS_PRONE,	 0,  LOG_NORMAL, CMD_BREAKS_HIDE|CMD_BREAKS_PREDELAY	},
    { "close",		do_close,	POS_PRONE,	 0,  LOG_NORMAL, CMD_BREAKS_HIDE|CMD_BREAKS_PREDELAY	},
	{ "disarm",		do_disarm,	POS_PRONE,	 0,  LOG_NORMAL, CMD_BREAKS_HIDE|CMD_BREAKS_PREDELAY	},
	{ "drag",		do_drag,	POS_PRONE,	 0,  LOG_NORMAL, CMD_BREAKS_HIDE|CMD_BREAKS_PREDELAY	},
    { "drink",		do_drink,	POS_PRONE,	 0,  LOG_NORMAL, CMD_BREAKS_HIDE|CMD_BREAKS_PREDELAY	},
    { "drop",		do_drop,	POS_PRONE,	 0,  LOG_NORMAL, CMD_BREAKS_HIDE|CMD_BREAKS_PREDELAY	},
    { "eat",		do_eat,		POS_PRONE,	 0,  LOG_NORMAL, CMD_BREAKS_HIDE|CMD_BREAKS_PREDELAY	},
    { "fill",		do_fill,	POS_PRONE,	 0,  LOG_NORMAL, CMD_BREAKS_HIDE|CMD_BREAKS_PREDELAY	},
    { "give",		do_give,	POS_PRONE,	 0,  LOG_NORMAL, CMD_BREAKS_HIDE|CMD_BREAKS_PREDELAY	},
	{ "knock",		do_knock,	POS_PRONE,	 0,  LOG_NORMAL, CMD_BREAKS_HIDE|CMD_BREAKS_PREDELAY	},
    //{ "hold",		do_group,	POS_PRONE,	 0,  LOG_NORMAL, CMD_BREAKS_HIDE|CMD_BREAKS_PREDELAY	},
	{ "invoke",		do_invoke,	POS_PRONE,	 0,  LOG_NORMAL, CMD_BREAKS_HIDE|CMD_BREAKS_PREDELAY	},
    //{ "list",		do_list,	POS_PRONE,	 0,  LOG_NORMAL, 0	},
    { "lock",		do_lock,	POS_PRONE,	 0,  LOG_NORMAL, CMD_BREAKS_HIDE|CMD_BREAKS_PREDELAY	},
    { "open",		do_open,	POS_PRONE,	 0,  LOG_NORMAL, CMD_BREAKS_HIDE|CMD_BREAKS_PREDELAY	},
    { "pick",		do_pick,	POS_PRONE,	 0,  LOG_NORMAL, CMD_BREAKS_HIDE|CMD_BREAKS_PREDELAY	},
    { "put",		do_newput,	POS_PRONE,	 0,  LOG_NORMAL, CMD_BREAKS_HIDE|CMD_BREAKS_PREDELAY },
    { "remove",		do_remove,	POS_PRONE,	 0,  LOG_NORMAL, CMD_BREAKS_HIDE|CMD_BREAKS_PREDELAY	},
	{ "rub",		do_rub,		POS_PRONE,	 0,  LOG_NORMAL, CMD_BREAKS_HIDE|CMD_BREAKS_PREDELAY	},
    { "sell",		do_sell2,	POS_PRONE,	 0,  LOG_NORMAL, CMD_BREAKS_HIDE|CMD_BREAKS_PREDELAY	},
	{ "show",		do_show,	POS_PRONE,	 0,  LOG_NORMAL, CMD_BREAKS_HIDE|CMD_BREAKS_PREDELAY	},
	{ "skin",		do_skin,	POS_PRONE,	 0,  LOG_NORMAL, CMD_BREAKS_HIDE|CMD_BREAKS_PREDELAY	},
	{ "swap",		do_swap,	POS_PRONE,	 0,  LOG_NORMAL, CMD_BREAKS_HIDE|CMD_BREAKS_PREDELAY	},
    { "take",		do_get,		POS_PRONE,	 0,  LOG_NORMAL, CMD_BREAKS_HIDE|CMD_BREAKS_PREDELAY	},
	{ "tend",		do_tend,	POS_PRONE,	 0,  LOG_NORMAL, CMD_BREAKS_HIDE|CMD_BREAKS_PREDELAY	},
    { "unlock",		do_unlock,	POS_PRONE,	 0,  LOG_NORMAL, CMD_BREAKS_HIDE|CMD_BREAKS_PREDELAY	},
	{ "wave",		do_wave,	POS_PRONE,	 0,  LOG_NORMAL, CMD_BREAKS_HIDE|CMD_BREAKS_PREDELAY	},
    { "wear",		do_wear,	POS_PRONE,	 0,  LOG_NORMAL, CMD_BREAKS_HIDE|CMD_BREAKS_PREDELAY	},

    /*
     * Combat commands.
     */

    /*
     * Miscellaneous commands.
     */
	{ "balance",		do_balance,	POS_PRONE,	 0,  LOG_NORMAL, CMD_BREAKS_HIDE|CMD_BREAKS_PREDELAY	},
	{ "check",		do_balance,	POS_PRONE,	 0,  LOG_NORMAL, CMD_BREAKS_HIDE|CMD_BREAKS_PREDELAY	},
	{ "deposit",		do_deposit,	POS_PRONE,	 0,  LOG_NORMAL, CMD_BREAKS_HIDE|CMD_BREAKS_PREDELAY	}, 
	{ "withdraw",		do_withdraw,	POS_PRONE,	 0,  LOG_NORMAL, CMD_BREAKS_HIDE|CMD_BREAKS_PREDELAY	},
	 
	{ "depart",		do_depart,	POS_PRONE,	 0,  LOG_NORMAL, BYPASS_DEAD	},
	{ "exi",		do_exi,		POS_PRONE,	 0,  LOG_NORMAL, BYPASS_STUN|BYPASS_RT|BYPASS_DEAD	},
    { "exit",		do_exit,	POS_PRONE,	 0,  LOG_NORMAL, BYPASS_STUN|BYPASS_RT|BYPASS_DEAD	},
    { "join",		do_follow,	POS_PRONE,	 0,  LOG_NORMAL, CMD_BREAKS_HIDE|CMD_BREAKS_PREDELAY	},
	{ "follow",		do_follow,	POS_PRONE,	 0,  LOG_NORMAL, CMD_BREAKS_HIDE|CMD_BREAKS_PREDELAY	},
    //{ "group",		do_group,	POS_PRONE,	 0,  LOG_NORMAL, CMD_BREAKS_HIDE|CMD_BREAKS_PREDELAY	},
    //{ "hold",		do_group,	POS_PRONE,	 0,  LOG_NORMAL, CMD_BREAKS_HIDE|CMD_BREAKS_PREDELAY	},
    { "hide",		do_hide,	POS_PRONE,	 0,  LOG_NORMAL, 0	},
	// { "increase",	do_train,	POS_PRONE,	 0,  LOG_NORMAL, 0	},
	{ "loot",		do_loot,	POS_PRONE,	 0,  LOG_NORMAL, CMD_BREAKS_HIDE|CMD_BREAKS_PREDELAY	},
    { "qui",		do_qui,		POS_PRONE,	 0,  LOG_NORMAL, BYPASS_STUN|BYPASS_RT|BYPASS_DEAD	},
    { "quit",		do_quit,	POS_PRONE,	 0,  LOG_NORMAL, BYPASS_STUN|BYPASS_RT|BYPASS_DEAD	},
    { "release",	do_release,	POS_PRONE,	 0,  LOG_NORMAL, 0	},
    //{ "rent",		do_rent,	POS_PRONE,	 0,  LOG_NORMAL, BYPASS_STUN|BYPASS_RT	},
    { "save",		do_save,	POS_PRONE,	 0,  LOG_NORMAL, BYPASS_DEAD|BYPASS_STUN|BYPASS_RT	},
	{ "search",		do_search,	POS_PRONE,	 0,  LOG_NORMAL, CMD_BREAKS_HIDE|CMD_BREAKS_PREDELAY	},
	{ "skills",		do_skills,	POS_PRONE,	 0,  LOG_NORMAL, BYPASS_DEAD|BYPASS_STUN|BYPASS_RT	},
	{ "sneak",		do_sneak,	POS_PRONE,	 0,  LOG_NORMAL, 0	},
    { "share",		do_split,	POS_PRONE,	 0,  LOG_NORMAL, CMD_BREAKS_HIDE|CMD_BREAKS_PREDELAY	},
    { "steal",		do_steal,	POS_PRONE,	 0,  LOG_NORMAL, CMD_BREAKS_HIDE|CMD_BREAKS_PREDELAY	}, // will break hide on its own
    { "train",		do_practice,POS_PRONE,	 0,  LOG_NORMAL, 0	},
    { "unhide",	    do_unhide,	POS_PRONE,	 0,  LOG_NORMAL, CMD_BREAKS_PREDELAY }, // breaks hide on its own
	{ "visible",	do_unhide,	POS_PRONE,	 0,  LOG_NORMAL, CMD_BREAKS_PREDELAY }, // breaks hide on its own
    { "wake",		do_wake,	POS_PRONE,	 0,  LOG_NORMAL, 0	},
	
		//char gen

    { "choose",		do_select,	POS_PRONE,	 0,  LOG_NORMAL, BYPASS_STUN|BYPASS_RT|CMD_DONT_PARSE		},
	{ "begin",		do_begin,	POS_PRONE,	 0,  LOG_NORMAL, BYPASS_STUN|BYPASS_RT|CMD_DONT_PARSE		},	
	//{ "choose",		do_choose,	POS_PRONE,	 0,  LOG_NORMAL, BYPASS_STUN|BYPASS_RT|CMD_DONT_PARSE		},	
    { "select",		do_select,	POS_PRONE,	 0,  LOG_NORMAL, BYPASS_STUN|BYPASS_RT|CMD_DONT_PARSE		},
	{ "roll",		do_roll,	POS_PRONE,	 0,  LOG_NORMAL, BYPASS_STUN|BYPASS_RT|CMD_DONT_PARSE		},
	{ "rename",		do_rename,	POS_PRONE,	 0,  LOG_NORMAL, BYPASS_STUN|BYPASS_RT|CMD_DONT_PARSE		},
	
	{ "age",	do_age,	    POS_PRONE,	 0,  LOG_NORMAL, BYPASS_STUN|BYPASS_RT|CMD_DONT_PARSE	},
    { "eyes",	do_eyes,	POS_PRONE,	 0,  LOG_NORMAL, BYPASS_STUN|BYPASS_RT|CMD_DONT_PARSE	},
	{ "hair",	do_hair,	POS_PRONE,	 0,  LOG_NORMAL, BYPASS_STUN|BYPASS_RT|CMD_DONT_PARSE	},
	{ "skincolor",	do_skincolor,	POS_PRONE,	 0,  LOG_NORMAL, BYPASS_STUN|BYPASS_RT|CMD_DONT_PARSE	},

// no parse isn't strictly necessary but parsing is never needed in OLC
	
	// olc commands
	{ "hset",		do_hset,	POS_PRONE,	MAX_LEVEL,  LOG_ALWAYS, COMMAND_OLC|BYPASS_STUN|BYPASS_RT|CMD_DONT_PARSE },
	{ "newhelp",		do_newhelp,	POS_PRONE,	MAX_LEVEL,  LOG_ALWAYS, COMMAND_OLC|BYPASS_STUN|BYPASS_RT|CMD_DONT_PARSE },
	{ "savehelps",		do_savehelps,	POS_PRONE,	MAX_LEVEL,  LOG_ALWAYS, COMMAND_OLC|BYPASS_STUN|BYPASS_RT|CMD_DONT_PARSE },
	{ "savezone",		do_savezone,	POS_PRONE,	MAX_LEVEL,  LOG_ALWAYS, COMMAND_OLC|BYPASS_STUN|BYPASS_RT|CMD_DONT_PARSE },	
	{ "break",		do_break,	POS_PRONE,	MAX_LEVEL,  LOG_ALWAYS, COMMAND_OLC|BYPASS_STUN|BYPASS_RT|CMD_DONT_PARSE },
	{ "collaps",		do_collaps,	POS_PRONE,	MAX_LEVEL,  LOG_ALWAYS, COMMAND_OLC|BYPASS_STUN|BYPASS_RT|CMD_DONT_PARSE },
	{ "collapse",		do_collapse,	POS_PRONE,	MAX_LEVEL,  LOG_ALWAYS, COMMAND_OLC|BYPASS_STUN|BYPASS_RT|CMD_DONT_PARSE },
	{ "connect",		do_connect,	POS_PRONE,	MAX_LEVEL,  LOG_ALWAYS, COMMAND_OLC|BYPASS_STUN|BYPASS_RT|CMD_DONT_PARSE },
	{ "dig",		do_dig,	POS_PRONE,	MAX_LEVEL,  LOG_ALWAYS, COMMAND_OLC|BYPASS_STUN|BYPASS_RT|CMD_DONT_PARSE },
	{ "exset",		do_exset,	POS_PRONE,	MAX_LEVEL,  LOG_ALWAYS, COMMAND_OLC|BYPASS_STUN|BYPASS_RT },
	{ "rdesc",		do_rdesc,	POS_PRONE,	MAX_LEVEL,  LOG_ALWAYS, COMMAND_OLC|BYPASS_STUN|BYPASS_RT|CMD_DONT_PARSE },
	{ "rextra",		do_rextra,	POS_PRONE,	MAX_LEVEL,  LOG_ALWAYS, COMMAND_OLC|BYPASS_STUN|BYPASS_RT },
	{ "ddesc",		do_ddesc,	POS_PRONE,	MAX_LEVEL,  LOG_ALWAYS, COMMAND_OLC|BYPASS_STUN|BYPASS_RT|CMD_DONT_PARSE },
	
	
	
	
    /*
     * Immortal commands.
     */
	{ "parse",		do_parse,	POS_PRONE,	MAX_LEVEL,  LOG_ALWAYS, BYPASS_STUN|BYPASS_RT },
	{ "commands",	do_commands,	POS_PRONE,	 MAX_LEVEL,  LOG_NORMAL, BYPASS_STUN|BYPASS_RT },
	{ "wizhelp",	do_wizhelp,	POS_PRONE,	 MAX_LEVEL,  LOG_NORMAL, BYPASS_STUN|BYPASS_RT },	 
	{ "socials",	do_socials,	POS_PRONE,	 MAX_LEVEL,  LOG_NORMAL, BYPASS_STUN|BYPASS_RT },
	
    { "advance",	do_advance,	POS_PRONE,	MAX_LEVEL,  LOG_ALWAYS, BYPASS_STUN|BYPASS_RT },
    { "trust",		do_trust,	POS_PRONE,	MAX_LEVEL,  LOG_ALWAYS, BYPASS_STUN|BYPASS_RT },

    { "allow",		do_allow,	POS_PRONE,	MAX_LEVEL-1,  LOG_ALWAYS, BYPASS_STUN|BYPASS_RT },
    { "ban",		do_ban,		POS_PRONE,	MAX_LEVEL-1,  LOG_ALWAYS, BYPASS_STUN|BYPASS_RT },
    { "deny",		do_deny,	POS_PRONE,	MAX_LEVEL-1,  LOG_ALWAYS, BYPASS_STUN|BYPASS_RT },
    { "disconnect",	do_disconnect,	POS_PRONE,	MAX_LEVEL-1,  LOG_ALWAYS, BYPASS_STUN|BYPASS_RT },
    { "freeze",		do_freeze,	POS_PRONE,	MAX_LEVEL-1,  LOG_ALWAYS, BYPASS_STUN|BYPASS_RT },
    { "reboo",		do_reboo,	POS_PRONE,	MAX_LEVEL-1,  LOG_NORMAL, BYPASS_STUN|BYPASS_RT },
    { "reboot",		do_reboot,	POS_PRONE,	MAX_LEVEL-1,  LOG_ALWAYS, BYPASS_STUN|BYPASS_RT },
    { "shutdow",	do_shutdow,	POS_PRONE,	MAX_LEVEL-1,  LOG_NORMAL, BYPASS_STUN|BYPASS_RT },
    { "shutdown",	do_shutdown,	POS_PRONE,	MAX_LEVEL-1,  LOG_ALWAYS, BYPASS_STUN|BYPASS_RT },
    { "users",		do_users,	POS_PRONE,	MAX_LEVEL-1,  LOG_NORMAL, BYPASS_STUN|BYPASS_RT },
    { "wizlock",	do_wizlock,	POS_PRONE,	MAX_LEVEL-1,  LOG_ALWAYS, BYPASS_STUN|BYPASS_RT },

	{ "rload",		do_rload,	POS_PRONE,	MAX_LEVEL-2,  LOG_ALWAYS, BYPASS_STUN|BYPASS_RT },
	{ "cload",		do_cload,	POS_PRONE,	MAX_LEVEL-2,  LOG_ALWAYS, BYPASS_STUN|BYPASS_RT },
    { "force",		do_force,	POS_PRONE,	MAX_LEVEL-2,  LOG_ALWAYS, BYPASS_STUN|BYPASS_RT },
    { "mload",		do_mload,	POS_PRONE,	MAX_LEVEL-2,  LOG_ALWAYS, BYPASS_STUN|BYPASS_RT },
    { "mset",		do_mset,	POS_PRONE,	MAX_LEVEL-2,  LOG_ALWAYS, BYPASS_STUN|BYPASS_RT },
    { "noemote",	do_noemote,	POS_PRONE,	MAX_LEVEL-2,  LOG_NORMAL, BYPASS_STUN|BYPASS_RT },
    { "notell",		do_notell,	POS_PRONE,	MAX_LEVEL-2,  LOG_NORMAL, BYPASS_STUN|BYPASS_RT },
    { "oload",		do_oload,	POS_PRONE,	MAX_LEVEL-2,  LOG_ALWAYS, BYPASS_STUN|BYPASS_RT },
    { "oset",		do_oset,	POS_PRONE,	MAX_LEVEL-2,  LOG_ALWAYS, BYPASS_STUN|BYPASS_RT },
    { "pardon",		do_pardon,	POS_PRONE,	MAX_LEVEL-2,  LOG_ALWAYS, BYPASS_STUN|BYPASS_RT },
    { "purge",		do_purge,	POS_PRONE,	MAX_LEVEL-2,  LOG_NORMAL, BYPASS_STUN|BYPASS_RT },
    { "restore",	do_restore,	POS_PRONE,	MAX_LEVEL-2,  LOG_ALWAYS, BYPASS_STUN|BYPASS_RT },
    { "rset",		do_rset,	POS_PRONE,	MAX_LEVEL-2,  LOG_ALWAYS, BYPASS_STUN|BYPASS_RT },
    { "silence",	do_silence,	POS_PRONE,	MAX_LEVEL-2,  LOG_NORMAL, BYPASS_STUN|BYPASS_RT },
    { "sla",		do_sla,		POS_PRONE,	MAX_LEVEL-2,  LOG_NORMAL, BYPASS_STUN|BYPASS_RT },
    { "slay",		do_slay,	POS_PRONE,	MAX_LEVEL-2,  LOG_ALWAYS, BYPASS_STUN|BYPASS_RT },
    { "sset",		do_sset,	POS_PRONE,	MAX_LEVEL-2,  LOG_ALWAYS, BYPASS_STUN|BYPASS_RT },
    { "transfer",	do_transfer,	POS_PRONE,	MAX_LEVEL-2,  LOG_ALWAYS, BYPASS_STUN|BYPASS_RT },
    { "at",		do_at,		POS_PRONE,	MAX_LEVEL-3,  LOG_NORMAL, BYPASS_STUN|BYPASS_RT|CMD_DONT_PARSE },
    { "bamfin",		do_bamfin,	POS_PRONE,	MAX_LEVEL-3,  LOG_NORMAL, BYPASS_STUN|BYPASS_RT },
    { "bamfout",	do_bamfout,	POS_PRONE,	MAX_LEVEL-3,  LOG_NORMAL, BYPASS_STUN|BYPASS_RT },
    { "echo",		do_echo,	POS_PRONE,	MAX_LEVEL-3,  LOG_ALWAYS, BYPASS_STUN|BYPASS_RT|CMD_DONT_PARSE },
    { "goto",		do_goto,	POS_PRONE,	MAX_LEVEL-3,  LOG_NORMAL, BYPASS_STUN|BYPASS_RT },
    { "holylight",	do_holylight,	POS_PRONE,	MAX_LEVEL-3,  LOG_NORMAL, BYPASS_STUN|BYPASS_RT },
    { "invis",		do_invis,	POS_PRONE,	MAX_LEVEL-3,  LOG_NORMAL, BYPASS_STUN|BYPASS_RT },
    { "log",		do_log,		POS_PRONE,	MAX_LEVEL-3,  LOG_ALWAYS, BYPASS_STUN|BYPASS_RT },
    { "memory",		do_memory,	POS_PRONE,	MAX_LEVEL-3,  LOG_NORMAL, BYPASS_STUN|BYPASS_RT },
    { "mfind",		do_mfind,	POS_PRONE,	MAX_LEVEL-3,  LOG_NORMAL, BYPASS_STUN|BYPASS_RT },
    { "mstat",		do_mstat,	POS_PRONE,	MAX_LEVEL-3,  LOG_NORMAL, BYPASS_STUN|BYPASS_RT },
    { "mwhere",		do_mwhere,	POS_PRONE,	MAX_LEVEL-3,  LOG_NORMAL, BYPASS_STUN|BYPASS_RT },
    { "ofind",		do_ofind,	POS_PRONE,	MAX_LEVEL-3,  LOG_NORMAL, BYPASS_STUN|BYPASS_RT },
    { "ostat",		do_ostat,	POS_PRONE,	MAX_LEVEL-3,  LOG_NORMAL, BYPASS_STUN|BYPASS_RT },
    { "peace",		do_peace,	POS_PRONE,	MAX_LEVEL-3,  LOG_NORMAL, BYPASS_STUN|BYPASS_RT },
	{ "randshield",		do_randshield,	POS_PRONE,	MAX_LEVEL-3,  LOG_NORMAL, BYPASS_STUN|BYPASS_RT },
	{ "randarmor",		do_randarmor,	POS_PRONE,	MAX_LEVEL-3,  LOG_NORMAL, BYPASS_STUN|BYPASS_RT },
    { "randbox",		do_randbox,	POS_PRONE,	MAX_LEVEL-3,  LOG_NORMAL, BYPASS_STUN|BYPASS_RT },
	{ "randcon",		do_randcontainer,	POS_PRONE,	MAX_LEVEL-3,  LOG_NORMAL, BYPASS_STUN|BYPASS_RT },
	{ "randgem",		do_randgem,	POS_PRONE,	MAX_LEVEL-3,  LOG_NORMAL, BYPASS_STUN|BYPASS_RT },
    { "randweapon",		do_randweapon,	POS_PRONE,	MAX_LEVEL-3,  LOG_ALWAYS, BYPASS_STUN|BYPASS_RT },
	{ "randmagic",		do_randmagic,	POS_PRONE,	MAX_LEVEL-3,  LOG_ALWAYS, BYPASS_STUN|BYPASS_RT },
    { "recho",		do_recho,	POS_PRONE,	MAX_LEVEL-3,  LOG_ALWAYS, BYPASS_STUN|BYPASS_RT },
    { "return",		do_return,	POS_PRONE,	MAX_LEVEL-3,  LOG_NORMAL, BYPASS_STUN|BYPASS_RT },
    { "rstat",		do_rstat,	POS_PRONE,	MAX_LEVEL-3,  LOG_NORMAL, BYPASS_STUN|BYPASS_RT },
    { "slookup",	do_slookup,	POS_PRONE,	MAX_LEVEL-3,  LOG_NORMAL, BYPASS_STUN|BYPASS_RT },
    { "snoop",		do_snoop,	POS_PRONE,	MAX_LEVEL-3,  LOG_NORMAL, BYPASS_STUN|BYPASS_RT },
    { "switch",		do_switch,	POS_PRONE,	MAX_LEVEL-3,  LOG_ALWAYS, BYPASS_STUN|BYPASS_RT },
	{ "viewskills",		do_viewskills,	POS_PRONE,	MAX_LEVEL-3,  LOG_ALWAYS, BYPASS_STUN|BYPASS_RT },

    { "immtalk",	do_immtalk,	POS_PRONE,	MAX_LEVEL-3,  LOG_NORMAL, BYPASS_STUN|BYPASS_RT|CMD_DONT_PARSE },
    { ":",		do_immtalk,	POS_PRONE,	MAX_LEVEL-3,  LOG_NORMAL, BYPASS_STUN|BYPASS_RT|CMD_DONT_PARSE },

    /*
     * End of list.
     */
    { "",		0,		POS_PRONE,	 0,  LOG_NORMAL, 0	}
};



/*
 * The social table.
 * Add new socials here.
 * Alphabetical order is not required.
 */
const	struct	social_type	social_table [] =
{
    {
	"accuse",
	"Accuse whom?",
	"$n is in an accusing mood.",
	"You look accusingly at $M.",
	"$n looks accusingly at $N.",
	"$n looks accusingly at you.",
	"You accuse yourself.",
	"$n seems to have a bad conscience.",
	"You accuses the inferior nature of $s $o to be the problem.",
	"$n accuses the inferior nature $s $o to be the problem.",
	"You accuse $p of being out of order.",
	"$n accuses $p of being out of order."
    },

    {
	"applaud",
	"Clap, clap, clap.",
	"$n gives a round of applause.",
	"You clap at $S actions.",
	"$n claps at $N's actions.",
	"$n gives you a round of applause.",
	"You applaud at yourself.",
	"$n applauds at $mself.",
	"You celebrate $p in your hand.",
	"$n celebrates $p in $s hand.",
	"You really appreciate $p.",
	"$n really appreciates $p."
    },

    {
	"bark",
	"Woof!  Woof!",
	"$n barks like a dog.",
	"You bark at $M.",
	"$n barks at $N.",
	"$n barks at you.",
	"You bark at yourself.",
	"$n barks at $mself.",
	"You bark at your $o.",
	"$n bark at $s $o.",
	"You bark at $p.",
	"$n barks at $p."
    },

    {
	"blush",
	"Your cheeks are burning.",
	"$n blushes.",
	"You get all flustered up seeing $M.",
	"$n blushes as $e sees $N here.",
	"$n blushes as $e sees you here.",
	"You blush at your own folly.",
	"$n blushes, quite embarassed.",
	"You blush at your $o.",
	"$n blush at $s $o.",
	"You blushes at the sight of $p.",
	"$n blushes at the sight of $p."
    },

    {
	"bounce",
	"You bounce around.",
	"$n bounces around.",
	"You bounce into $M!",
	"$n bounces into $N!",
	"$n bounces into you!",
	"You bounce up and down.",
	"$n bounces up and down.",
	"You look at your $o and bounces up and down.",
	"$n looks at $s $o at bounces up and down.",
	"You bounces violently against $p!",
	"$n bounce violently against $p!"
    },

    {
	"bow",
	"You bow.",
	"$n bows.",
	"You bow to $M.",
	"$n bows to $N.",
	"$n bows to you.",
	"You bow deeply.",
	"$n bows deeply.",
	"You bow, holding your $o ceremoniously.",
	"$n bow, holding $s $o ceremoniously.",
	"You bows before $p.",
	"$n bows before $p."
    },

    {
	"burp",
	"You burp loudly.",
	"$n burps loudly.",
	"You burp loudly in $N's direction.",
	"$n burps loudly in $N's direction.",
	"$n burps loudly your direction.",
	"You burp quietly.",
	"$n burps quietly.",
	"You burp at your $o.",
	"$n burp at $s $o.",
	"You burps at $p.",
	"$n burps at $p."
    },

    {
	"cackle",
	"You cackle!",
	"$n cackles!",
	"You cackle at $N!",
	"$n cackles at $N!",
	"$n cackles at you!",
	"You cackle madly!",
	"$n cackles madly!",
	"You cackle at $p in $s inventory.",
	"$n cackle at $p in $s inventory.",
	"You cackles at $p in the room.",
	"$n cackles at $p in the room."
    },

    {
	"chuckle",
	"You chuckle.",
	"$n chuckles.",
	"You chuckle at $N.",
	"$n chuckles at $N.",
	"$n chuckles at you.",
	"You chuckle at your own joke.",
	"$n chuckles at $s own joke.",
	"You chuckle at $p in $s inventory.",
	"$n chuckle at $p in $s inventory.",
	"You chuckles at $p in the room.",
	"$n chuckle at $p in the room."
    },

    {
	"clap",
	"You clap your hands together.",
	"$n shows $s approval by clapping $s hands together.",
	"You clap at $S performance.",
	"$n claps at $N's performance.",
	"$n claps at your performance.",
	"You clap at your own performance.",
	"$n claps at $s own performance.",
	"You try to clap for $p but it gets in the way in your hand.",
	"$n tries to clap for $p but it gets in the way in $s hand.",
	"You clap for $p.",
	"$n claps for $p."
    },

    {
	"cry",
	"You burst into tears.",
	"$n bursts into tears.",
	"You cry on $S shoulder.",
	"$n cries on $N's shoulder.",
	"$n cries on your shoulder.",
	"You cry to yourself.",
	"$n sobs quietly to $mself.",
	"You cry at the sight of $p.",
	"$n cries at the sight of $p.",
	"You cry at the sight of $p.",
	"$n cry at the sight of $p."
    },

    {
	"curse",
	"You swear loudly for a long time.",
	"$n swears: @*&^%@*&!",
	"You swear at $M.",
	"$n swears at $N.",
	"$n swears at you!  Where are $s manners?",
	"You swear at your own mistakes.",
	"$n starts swearing at $mself.  Why don't you help?",
	"You curse at your $o.",
	"$n curses at $s $o.",
	"You curse at $p.",
	"$n curses at $p."
    },

    {
	"curtsey",
	"You curtsey to your audience.",
	"$n curtseys gracefully.",
	"You curtsey to $N.",
	"$n curtseys to $N.",
	"$n curtseys to you.",
	"You curtsey shyly.",
	"$n curtseys shyly.",
	"You curtsey at your $o.",
	"$n curtsey at $s $o.",
	"You curtseys at $p.",
	"$n curtseys at $p."
    },

    {
	"dance",
	"You dance around.",
	"$n dances around.",
	"You invite $N to dance with you.",
	"$n invites $N to dance with $m.",
	"$n invites $N to dance with $m.",
	"You skip and dance around by yourself.",
	"$n skip and dance around by $mself.",
	"You do a little dance, raising your $o.",
	"$n does a little dance, raising $s $o.",
	"You do a little dance at the sight of at $p.",
	"$n does a little dance at the sight of at $p."
    },

    {
	"drool",
	"You drool on yourself.",
	"$n drools on $mself.",
	"You drool all over $N.",
	"$n drools all over $N.",
	"$n drools all over you.",
	"You drool on yourself.",
	"$n drools on $mself.",
	"You drool at your $o.",
	"$n drools at $s $o.",
	"You drools at $p in the room.",
	"$n drools at $p in the room."
    },

    {
	"frown",
	"What's bothering you ?",
	"$n frowns.",
	"You frown at what $E did.",
	"$n frowns at what $E did.",
	"$n frowns at what you did.",
	"You frown at yourself.",
	"$n frowns at $mself.",
	"You frown at your $o.",
	"$n frowns at $s $o.",
	"You frown at $p in the room.",
	"$n frowns at $p in the room."
    },

    {
	"gasp",
	"You gasp.",
	"$n gasps.",
	"You gasp.",
	"$n gasps.",
	"$n gasps.",
	"You gasp at yourself.",
	"$n gasp at $mself.",
	"You gasp at your $o.",
	"$n gasps at $s $o.",
	"You gasp at $p in the room.",
	"$n gasps at $p in the room."
    },

    {
	"giggle",
	"You giggle.",
	"$n giggles.",
	"You giggle at $N.",
	"$n giggles at $N.",
	"$n giggles at you.",
	"You giggle at yourself.",
	"$n giggles at $mself.",
	"You VERB at $p in $s inventory.",
	"$n VERB at $p in $s inventory.",
	"You VERBs at $p in the room.",
	"$n VERBs at $p in the room."
    },

    {
	"glare",
	"You glare at nothing in particular.",
	"$n glares around $m.",
	"You glare at $M.",
	"$n glares at $N.",
	"$n glares at you.",
	"You glower.",
	"$n glowers.",
	"You VERB at $p in $s inventory.",
	"$n VERB at $p in $s inventory.",
	"You VERBs at $p in the room.",
	"$n VERBs at $p in the room."
    },

    {
	"grin",
	"You grin.",
	"$n grins.",
	"You grin at $M.",
	"$n grins at $N.",
	"$n grins at you.",
	"You grin to yourself.",
	"$n grins to $mself.",
	"You grin at your $o.",
	"$n grins at $s $o.",
	"You grin at $p.",
	"$n grins at $p."
    },

    {
	"groan",
	"You groan loudly.",
	"$n groans loudly.",
	"You groan at the sight of $M.",
	"$n groans at the sight of $N.",
	"$n groans at the sight of you.",
	"You groan as you realize what you have done.",
	"$n groans as $e realizes what $e has done.",
	"You groan at your $o.",
	"$n groans at $s $o.",
	"You groan at $p.",
	"$n groan at $p."
    },

    {
	"growl",
	"Grrrrrrrrrr ...",
	"$n growls.",
	"Grrrrrrrrrr ... take that, $N!",
	"$n growls at $N.",
	"$n growls at you.",
	"You growl ferociously!",
	"$n growls ferociously!",
	NULL, NULL, NULL, NULL
    },

    {
	"grumble",
	"You grumble.",
	"$n grumbles.",
	"You grumble to $M.",
	"$n grumbles to $N.",
	"$n grumbles to you.",
	"You grumble under your breath.",
	"$n grumbles under $s breath.",
	"You grumble at your $p.",
	"$n grumbles at $s $p.",
	"You grumble at $p.",
	"$n grumbles at $p."
    },

    {
	"grunt",
	"You grunt.",
	"$n grunts.",
	NULL,
	NULL, NULL, NULL, NULL,
	NULL, NULL, NULL, NULL
    },

    {
	"hug",
	"Hug whom?",
	NULL,
	"You hug $M.",
	"$n hugs $N.",
	"$n hugs you.",
	"You hug yourself.",
	"$n hugs $mself in a vain attempt to get friendship.",
	"You VERB at $p in $s inventory.",
	"$n VERB at $p in $s inventory.",
	"You VERBs at $p in the room.",
	"$n VERBs at $p in the room."
    },

    {
	"kiss",
	"Isn't there someone you want to kiss?",
	NULL,
	"You kiss $M.",
	"$n kisses $N.",
	"$n kisses you.",
	"You think about kissing yourself, but think better of it.",
	NULL,
	"You kiss your $p.",
	"$n kisses $s $p.",
	"You gives $p a kiss.",
	"$n gives $p a kiss."
    },

    {
	"laugh",
	"You laugh!",
	"$n laughs!",
	"You laugh at $N!",
	"$n laughs at $N!",
	"$n laughs at you!",
	"You laugh softly to yourself.",
	"$n laughs softly to $mself.",
	"You look at your $p and laugh.",
	"$n looks at $s $p and laughs.",
	"You look at $p and laugh.",
	"$n looks at $p and laughs."
    },

    {
	"lick",
	"You lick your lips and smile.",
	"$n licks $s lips and smiles.",
	"You lick $M.",
	"$n licks $N.",
	"$n licks you.",
	"Oddly, you lick yourself.",
	"Oddly, $n licks $mself.",
	"You lick your $p.",
	"$n licks $s $p.",
	"You walk over to $p and lick it.",
	"$n walks over to $p and licks it."
    },


    {
	"moan",
	"You moan.",
	"$n moans.",
	"You moan at $N.",
	"$n moans at $N.",
	"$n moans at you.",
	"You moan loudly!",
	"$n moans loudly!",
	"You VERB at $p in $s inventory.",
	"$n VERB at $p in $s inventory.",
	"You VERBs at $p in the room.",
	"$n VERBs at $p in the room."
    },

  
    {
	"nod",
	"You nod.",
	"$n nods.",
	"You nod to $M.",
	"$n nods to $N.",
	"$n nods to you.",
	"You nod at yourself.",
	"$n nods at $mself.",
	"You nod at $p in $s inventory.",
	"$n nods at $p in $s inventory.",
	"You nod at $p in the room.",
	"$n nods at $p in the room."
    },

    {
	"nudge",
	"Nudge whom?",
	NULL,
	"You nudge $M.",
	"$n nudges $N.",
	"$n nudges you.",
	"You remind yoursef to get a move on.",
	"$n looks impatient.",
	"You bump your $o.",
	"$n bumps $s $o.",
	"You edge closer to $n.",
	"$n edge closer to $p."
    },

    {
	"pet",
	"Pet whom?",
	NULL,
	"You pet $N.",
	"$n pets $N.",
	"$n pets you.",
	"You pet yourself.",
	"$n pets $mself.",
	"You stroke $p gently.",
	"$n stroke $p gently.",
	"You run your fingers along $p.",
	"$n runs $s fingers along $p."
    },

    {
	"point",
	"Point at whom?",
	NULL,
	"You point at $M accusingly.",
	"$n points at $N accusingly.",
	"$n points at you accusingly.",
	"You point proudly at yourself.",
	"$n points proudly at $mself.",
	"You point at $p.",
	"$n points at $p.",
	"You point at $p.",
	"$n points at $p."
    },

    {
	"poke",
	"Poke whom?",
	NULL,
	"You poke $N.",
	"$n pokes $N.",
	"$n pokes you.",
	"You poke yourself.",
	"$n pokes $mself.",
	"You poke at your $o.",
	"$n pokes at $s o.",
	"You poke at $p.",
	"$n pokes at $p."
    },

    {
	"ponder",
	"You ponder.",
	"$n ponders.",
	NULL,
	NULL,
	NULL,
	NULL,
	NULL,
	NULL,
	NULL,
	NULL,
	NULL
    },

    {
	"pout",
	"Ah, don't take it so hard.",
	"$n pouts.",
	NULL,
	NULL,
	NULL,
	NULL,
	NULL,
	NULL,
	NULL,
	NULL,
	NULL
    },

    {
	"pray",
	"You pray.",
	"$n prays.",
	NULL,
	NULL,
	NULL,
	NULL,
	NULL,
	NULL,
	NULL,
	NULL,
	NULL
    },


    {
	"punch",
	"Punch whom?",
	NULL,
	"You punch $M playfully.",
	"$n punches $N playfully.",
	"$n punches you playfully.  OUCH!",
	"You punch yourself.  You deserve it.",
	"$n punches $mself.  Why don't you join in?",
	"You VERB at $p.",
	"$n VERB at $p.",
	"You VERB at $p.",
	"$n VERB at $p."
    },


    {
	"scream",
	"You scream!",
	"$n screams!",
	"You scream at $N!",
	"$n screams at $N!",
	"$n screams at you!",
	"You start to scream, but stifle yourself.",
	"$n looks very upset.",
	"You scream at $p!",
	"$n screams at $p!",
	"You scream at $p!",
	"$n screams at $p!"
    },

    {
	"shake",
	"You shake your head.",
	"$n shakes $s head.",
	"You shake $S hand.",
	"$n shakes $N's hand.",
	"$n shakes your hand.",
	"You are shaken by yourself.",
	"$n shakes and quivers like a bowl full of jelly.",
	"You shake your head at your $o.",
	"$n shakes $s head at at $s $o.",
	"You shake your head at $p.",
	"$n shakes $s head at $p."
    },

    {
	"shiver",
	"You shiver.",
	"$n shivers.",
	NULL,
	NULL,
	NULL,
	NULL,
	NULL,
	NULL,
	NULL,
	NULL,
	NULL
    },

    {
	"shrug",
	"You shrug.",
	"$n shrugs.",
	"You shrug at $N.",
	"$n shrugs at $N.",
	"$n shrugs at you.",
	NULL,
	NULL,
	NULL,
	NULL,
	NULL,
	NULL
    },

    {
	"sigh",
	"You sigh.",
	"$n sighs.",
	"You sigh at $N.",
	"$n sighs at $N.",
	"$n sighs as you.",
	"You sigh loudly.",
	"$n sighs loudly.",
	NULL,
	NULL,
	NULL,
	NULL
    },

    {
	"sing",
	"You raise your clear voice towards the sky.",
	"$n has begun to sing.",
	"You sing a ballad to $m.",
	"$n sings a ballad to $N.",
	"$n sings a ballad to you!  How sweet!",
	"You sing a little ditty to yourself.",
	"$n sings a little ditty to $mself.",
	"You VERB at $p.",
	"$n VERB at $p.",
	"You VERB at $p.",
	"$n VERB at $p."
    },

    {
	"smile",
	"You smile.",
	"$n smiles.",
	"You smile at $M.",
	"$n smile at $N.",
	"$n smiles at you.",
	"You smile.",
	"$n smiles.",
	"You smile at your $o.",
	"$n smiles at $s o.",
	"You smile at $p.",
	"$n smile at $p."
    },

    {
	"smirk",
	"You smirk.",
	"$n smirks.",
	"You smirk at $N.",
	"$n smirks at $N.",
	"$n smirks at you.",
	NULL,
	NULL,
	NULL,
	NULL,
	NULL,
	NULL
    },

    {
	"snap",
	"You snap your fingers.",
	"$n snaps $s fingers.",
	"You snap your fingers at $N.",
	"$n snaps $s fingers at $N.",
	"$n snaps $s fingers at you.",
	NULL,
	NULL,
	NULL,
	NULL,
	NULL,
	NULL
    },

    {
	"snarl",
	"You snarl!",
	"$n snarls!",
	"You snarl at $M!",
	"$n snarls at $N!",
	NULL,
	NULL,
	NULL,
	NULL,
	NULL,
	NULL,
	NULL
    },

    {
	"sneeze",
	"You sneeze.",
	"$n sneezes.",
	NULL,
	NULL,
	NULL,
	NULL,
	NULL,
	NULL,
	NULL,
	NULL,
	NULL
    },

    {
	"snicker",
	"You snicker.",
	"$n snickers.",
	"You snicker at $N.",
	"$n snickers at $N.",
	"$n snickers at you.",
	"You snicker to yourself.",
	"$n snickers to $mself.",
	NULL,
	NULL,
	NULL,
	NULL
    },

    {
	"sniff",
	"You sniff.",
	"$n sniffs.",
	NULL,
	NULL,
	NULL,
	NULL,
	NULL,
	NULL,
	NULL,
	NULL,
	NULL
    },

    {
	"snore",
	"Zzzzzzzzzzzzzzzzz.",
	"$n snores loudly.",
	NULL,
	NULL,
	NULL,
	NULL,
	NULL,
	"You snore at your $o.",
	"$n snore at $s $o.",
	"You snore at $p.",
	"$n snore at $p."
    },


    {
	"snuggle",
	"Who?",
	NULL,
	"you snuggle $M.",
	"$n snuggles up to $N.",
	"$n snuggles up to you.",
	"You snuggle up, getting ready to sleep.",
	"$n snuggles up, getting ready to sleep.",
	"You VERB at $p.",
	"$n VERB at $p.",
	"You VERB at $p.",
	"$n VERB at $p."
    },

    {
	"stare",
	"You stare off into space.",
	"$n stares off into space",
	"You stare at $N.",
	"$n stares at $N.",
	"$n stares at you.",
	NULL, NULL,
	"You stare at your $o.",
	"$n stares at $s $o.",
	"You stares at $p.",
	"$n stares at $p."
    },

    {
	"strut",
	"You strut around.",
	"$n struts around.",
	NULL,
	NULL,
	NULL,
	NULL,
	NULL,
	NULL,
	NULL,
	NULL,
	NULL,
    },

    {
	"sulk",
	"You sulk.",
	"$n sulks in the corner.",
	NULL,
	NULL,
	NULL,
	NULL,
	NULL,
	NULL,
	NULL,
	NULL,
	NULL,
    },

    {
	"thank",
	"Thank whom?",
	NULL,
	"You thank $N.",
	"$n thanks $N.",
	"$n thanks you.",
	NULL,
	NULL,
	NULL,
	NULL,
	NULL,
	NULL,
    },

    {
	"tickle",
	"Tickle whom?",
	NULL,
	"You tickle $N.",
	"$n tickles $N.",
	"$n tickles you.",
	"You tickle yourself.",
	"$n tickles $mself.",
	"You VERB at $p.",
	"$n VERB at $p.",
	"You VERB at $p.",
	"$n VERB at $p."
    },

    {
	"twiddle",
	"You twiddle your thumbs.",
	"$n twiddles $s thumbs.",
	NULL, NULL, NULL, NULL, NULL, NULL, NULL, NULL, NULL
    },

    {
	"wave",
	"You wave.",
	"$n waves.",
	"You wave at $N.",
	"$n waves at $N.",
	"$n waves at you.",
	"Going somewhere?",
	"$n waves at $mself.",
	"You VERB at $p.",
	"$n VERB at $p.",
	"You VERB at $p.",
	"$n VERB at $p."
    },

    {
	"whistle",
	"You whistle an idle bit of song.",
	"$n whistles an idle bit of song.",
	"You whistle at $M.",
	"$n whistles at $N.",
	"$n whistles at you.",
	"You whistle a little melody to yourself.",
	"$n whistles a little melody to $mself.",
	"You whistle at $p.",
	"$n whistle at $p.",
	"You VERB at $p.",
	"$n VERB at $p."
    },

    {
	"wince",
	"You wince.",
	"$n winces.",
	"You wince at $M.",
	"$n winces at $N.",
	"$n winces at you.",
	"You wince painfully.",
	"$n winces painfully.",
	"You wince at your $o.",
	"$n winces at $s $o.",
	"You wince at $p.",
	"$n winces at $p."
    },

    {
	"wink",
	"You wink.",
	"$n winks.",
	"You wink at $N.",
	"$n winks at $N.",
	"$n winks at you.",
	"You wink at yourself.",
	"$n winks at $mself.",
	"You wink at your $o.",
	"$n winks at $s $o.",
	"You wink at $p.",
	"$n winks at $p."
    },

    {
	"yawn",
	"You must be tired.",
	"$n yawns.",
	"You yawn at $N.",
	"$n yawns at $N.",
	"$n yawns at you.",
	"You yawn deeply.",
	"$n yawns deeply.",
	"You yawn at your $o.",
	"$n yawns at $s $o.",
	"You yawn at $p.",
	"$n yawns at $p."
    },

    {
	"",
	NULL, NULL, NULL, NULL, NULL, NULL, NULL,
	NULL, NULL, NULL, NULL
    }
};

bool add_roundtime( CHAR_DATA *ch, int npulse )
{
    ch->wait += npulse;
    return TRUE;
}

bool add_stun( CHAR_DATA *ch, int npulse )
{
    ch->stun += npulse;
    return TRUE;
}

void show_roundtime( CHAR_DATA *ch, int sec )
{
	printf_to_char( ch, "Roundtime: %d second%s.\r\n", sec,
	sec != 1 ? "s" : "" );
    return;
}

void show_stun( CHAR_DATA *ch, int sec )
{
	printf_to_char( ch, "You are stunned for %d second%s!\r\n", sec,
	sec != 1 ? "s" : "" );
    return;
}

bool check_specials( CHAR_DATA *ch, char *cmd, DO_FUN *fnptr, char *argument )
{
    CHAR_DATA *mob;
    OBJ_DATA *obj;
	
    for ( obj = ch->carrying; obj != NULL; obj = obj->next_content )
	{
	if ( obj->wear_loc != EQUIP_NONE  && obj->spec_fun != NULL )
	{
	    if ( (*obj->spec_fun) (ch, cmd, fnptr, argument, obj ) )
			return TRUE;
	}
	}
	
    for ( obj = ch->carrying; obj != NULL; obj = obj->next_content )
	{
	if ( obj->wear_loc == EQUIP_NONE && obj->spec_fun != NULL )
	{
	    if ( (*obj->spec_fun) (ch, cmd, fnptr, argument, obj ) )
			return TRUE;
	}
	}
	
    for ( obj = ch->in_room->contents; obj != NULL; obj = obj->next_content )
	{
	if ( obj->spec_fun != NULL )
	{
		if ( (*obj->spec_fun) (ch, cmd, fnptr, argument, obj ) ) 
			return TRUE;
	}
	}

	
    /* mobiles */
    for ( mob = ch->in_room->people; mob != NULL; mob = mob->next_in_room )
	{
	if ( mob->spec_fun != NULL )
	{
	    if ( (*mob->spec_fun) ( ch, cmd, fnptr, argument, mob ) )
		return TRUE;
	}
	}
	
    /* room */
    if ( ch->in_room->spec_fun != NULL )
	{
	if ( (*ch->in_room->spec_fun)( ch, cmd, fnptr, argument, ch->in_room ) )
		return TRUE;

	}
	
    return FALSE;
}


/*
 * The main entry point for executing commands.
 * Can be recursively called from 'at', 'order', 'force'.
 */
void interpret( CHAR_DATA *ch, char *argument )
{
    char command[MAX_INPUT_LENGTH];
    char logline[MAX_INPUT_LENGTH];
    int cmd;
    int trust;
    bool found;

    /*
     * Strip leading spaces.
     */
    while ( isspace((unsigned char)*argument) )
		argument++;
	
    if ( argument[0] == '\0' )
		return;


	// some commands break hide, not all
	

    /*
     * Implement freeze command.
     */
    if ( !IS_NPC(ch) && IS_SET(ch->act, PLR_FREEZE) )
    {
	send_to_char( "You're totally frozen!\r\n", ch );
	return;
    }	
	
	if ( !IS_AWAKE( ch ) )
    {
	send_to_char( "You are asleep.\r\n", ch );
	return;
    }

    /*
     * Grab the command word.
     * Special parsing so ' can be a command,
     *   also no spaces needed after punctuation.
     */
    strcpy( logline, argument );
    if ( !isalpha((unsigned char)argument[0]) && !isdigit((unsigned char)argument[0]) )
    {
	command[0] = argument[0];
	command[1] = '\0';
	argument++;
	while ( isspace((unsigned char)*argument) )
	    argument++;
    }
    else
    {
	argument = one_argument( argument, command );
    }

    /*
     * Look for command in command table.
     */
    found = FALSE;
    trust = get_trust( ch );
    for ( cmd = 0; cmd_table[cmd].name[0] != '\0'; cmd++ )
    {
	if ( command[0] == cmd_table[cmd].name[0]
	&&   !str_prefix( command, cmd_table[cmd].name )
	&&   cmd_table[cmd].level <= trust )
	{
	    found = TRUE;
	    break;
	}
    }

	if( !IS_NPC(ch) && IS_SET(ch->act, PLR_PARSE))
	{
		if( !IS_SET(cmd_table[cmd].flags, CMD_DONT_PARSE ) )
			argument = parse(argument,*cmd_table[cmd].do_fun);	
	}
	

	if( found )
	{
		
		
		if ( IS_SET( ch->affected_by, AFF_DEAD ) && !IS_SET(cmd_table[cmd].flags, BYPASS_DEAD) )
		{
		printf_to_char( ch, "You are dead.  You will decay in %d minute%s.\r\n\r\n", ch->decay < 60 ? 1 : ch->decay/ 60, ch->decay < 60 ? "" : "s"	);
		send_to_char( "(DEPART from your body if you wish to immediately decay.)\r\n\r\n", ch );
		//send_to_char( "(You will be required to CONFIRM.)\r\n\r\n", ch );
		return;
		}
	
        if ( ch->stun > 0 && !IS_SET(cmd_table[cmd].flags, BYPASS_STUN) )
	    {
	    printf_to_char( ch, "You are still stunned for another %d second%s.\r\n", ch->stun, ch->stun != 1
	    ? "s" : "" );
	    return;
	    }
		
		if ( ( IS_AFFECTED(ch, AFF_HOLD) || is_affected(ch, gsn_hold_undead) ) && !IS_SET(cmd_table[cmd].flags, BYPASS_RT) )
	    {
	    printf_to_char( ch, "You are held fast!\r\n" );
	    return;
	    }
	
		//fixme
        if ( ch->wait > 0 && !IS_SET(cmd_table[cmd].flags, BYPASS_RT) )
	    {
	    int sec = ch->wait / 1;

	    if ( (sec * 1) < ( ch->wait * 1 ) )
	    sec++;

	    if ( sec < 1 )
	    sec = 1;

	    printf_to_char( ch, "...wait %d second%s.\r\n", sec, sec != 1
	    ? "s" : "" );
	    return;
	    }
		
		if(  IS_AFFECTED(ch, AFF_HIDE) && IS_SET(cmd_table[cmd].flags, CMD_BREAKS_HIDE ) )
		{
		REMOVE_BIT( ch->affected_by, AFF_HIDE );
		act( "$n is revealed from hiding by $s action!", ch, NULL, NULL, TO_ROOM );
		send_to_char( "You are revealed from hiding by your action.\r\n", ch );

		}
		
		if( (IS_AFFECTED(ch, AFF_INVISIBLE) || is_affected(ch, gsn_invis) ) && IS_SET(cmd_table[cmd].flags, CMD_BREAKS_HIDE ) )
		{
		REMOVE_BIT   ( ch->affected_by, AFF_INVISIBLE	);
		affect_strip ( ch, gsn_invis			);
		act( "$n suddenly appears.", ch, NULL, NULL, TO_ROOM );
		send_to_char( "Your invisibilty fails.\r\n", ch );
		}
		
	/*
     * Lose concentration.
     */
    if ( IS_SET(cmd_table[cmd].flags, CMD_BREAKS_PREDELAY ) && ch->predelay_time > 0 )
    {
	if( ch->predelay_info->noInterrupt == FALSE )
	{
	send_to_char( "You stop concentrating on your task.\n\r", ch );
	ch->predelay_time = 0;
	free_predelay( ch->predelay_info );
	ch->predelay_info = NULL;
    }
	}


	}

    /*
     * Log and snoop.
     */
    if ( cmd_table[cmd].log == LOG_NEVER )
	strcpy( logline, "XXXXXXXX XXXXXXXX XXXXXXXX" );

    if ( ( !IS_NPC(ch) && IS_SET(ch->act, PLR_LOG) )
    ||   fLogAll
    ||   cmd_table[cmd].log == LOG_ALWAYS )
    {
	sprintf( log_buf, "Log %s: %s", ch->name, logline );
	log_string( log_buf );
    }

    if ( ch->desc != NULL && ch->desc->snoop_by != NULL )
    {
	write_to_buffer( ch->desc->snoop_by, "% ",    2 );
	write_to_buffer( ch->desc->snoop_by, logline, 0 );
	write_to_buffer( ch->desc->snoop_by, "\r\n",  2 );
    }

    /*
     * Character not in position for command?
     */
    if ( ch->position < cmd_table[cmd].position )
    {
	switch( ch->position )
	{
	case POS_PRONE:
	    send_to_char( "You cannot do that while you are lying down.\r\n", ch );
	    break;

	case POS_SITTING:
	    send_to_char( "You cannot do that while you are sitting down.\r\n", ch );
	    break;

	case POS_KNEELING:
	    send_to_char( "You cannot do that while you are kneeling.\r\n", ch);
	    break;

	}
	return;
    }
	
    if ( check_specials( ch, command, ( found ? *cmd_table[cmd].do_fun : NULL ), argument ) )
		return;
	
	// we have to put this above actual commands - what if the VT is 'rub' and 'rub' is also a command.
	// eventually check for object here so as to not nuke commands.
	//fixme add actual GSL to verb traps
	if ( check_verb_traps( ch, command, argument ) )
		return;
	

    /*
     * Dispatch the command.
     */
    if ( found )
    {
	(*cmd_table[cmd].do_fun) ( ch, argument );
    }
    else
    {
	if( command[0] == ';' )
	{
		    send_to_char( "But semicolons go in the middle, not the beginning.\n\r", ch );		
			return;
	}
	/*
	 * Look for command in socials table as the last check.
	 */ 
	 
	if ( !check_social( ch, command, argument ) )
	{
	    printf_to_char( ch, "Unrecognized input: %s %s\r\n", command, argument );
	}
	return;
    }

    tail_chain( );
    return;
}


bool check_verb_traps( CHAR_DATA *ch, char *command, char *argument )
{
	OBJ_DATA *obj;
	VERB_TRAP_DATA *vt;
	
	
	for ( obj = ch->in_room->contents; obj != NULL; obj = obj->next_content )
    {
		if ( IS_SET( obj->extra_flags, ITEM_VT_ROOM) && can_see_obj( ch, obj ) )
		{
			if( obj->verb_trap != NULL )
			{
				vt = obj->verb_trap;
				
				for ( ; vt != NULL; vt = vt->next )
				{
      
				if( command != NULL && vt->verb != NULL )
				{
				if ( is_name( command, vt->verb ) && obj == get_obj_here( ch, argument ) )
				{
					// this clues you in that there is a verb trap and the command is not invalid
				/*
				if ( !is_name( argument, obj->name ) )
				{
					printf_to_char(ch, "%s what?\r\n", capitalize(vt->verb) );
					return TRUE;
				}
				*/
					
					
				if( vt->firstPersonMessage != NULL )
				act( vt->firstPersonMessage, ch, obj, NULL, TO_CHAR );
			
				if( vt->roomMessage != NULL )
				act( vt->roomMessage, ch, obj, NULL, TO_ROOM );
			
				//fixme this will be actally made into a code, with a handler, rather than going straight to the parser
				if( vt->GSL != NULL  )
				interpret( ch, vt->GSL );
			
				return TRUE;
				}
				}
				}
			}
		}
    }
	
	for ( obj = ch->carrying; obj != NULL; obj = obj->next_content )
    {
		if ( IS_SET( obj->extra_flags, ITEM_VT_INVENTORY) && can_see_obj( ch, obj ) )
		{
			if( obj->verb_trap != NULL )
			{

				vt = obj->verb_trap;
				
				for ( ; vt != NULL; vt = vt->next )
				{

				if( command != NULL && vt->verb != NULL )
				{
				if ( is_name( command, vt->verb ) && is_name( argument, obj->name ) )
				{
					// this clues you in that there is a verb trap and the command is not invalid
				/*
				if ( !is_name( argument, obj->name ) )
				{
					printf_to_char(ch, "%s what?\r\n", capitalize(vt->verb) );
					return TRUE;
				}
				*/
					
					
				if( vt->firstPersonMessage != NULL )
				act( vt->firstPersonMessage, ch, obj, NULL, TO_CHAR );
			
				if( vt->roomMessage != NULL )
				act( vt->roomMessage, ch, obj, NULL, TO_ROOM );
			
				if( vt->GSL != NULL )
				interpret( ch, vt->GSL );
			
				return TRUE;
				}
				}
				}
			}
		}
    }
	
	return FALSE;
}


bool check_social( CHAR_DATA *ch, char *command, char *argument )
{
    char arg[MAX_INPUT_LENGTH];
    CHAR_DATA *victim;
    int cmd;
    bool found;

    found  = FALSE;
    for ( cmd = 0; social_table[cmd].name[0] != '\0'; cmd++ )
    {
	if ( command[0] == social_table[cmd].name[0]
	&&   !str_prefix( command, social_table[cmd].name ) )
	{
	    found = TRUE;
	    break;
	}
    }

    if ( !found )
	return FALSE;

    if ( !IS_NPC(ch) && IS_SET(ch->act, PLR_NO_EMOTE) )
    {
	send_to_char( "You are anti-social!\r\n", ch );
	return TRUE;
    }

    one_argument( argument, arg );
    victim = NULL;
    if ( arg[0] == '\0' )
    {
	act( social_table[cmd].others_no_arg, ch, NULL, victim, TO_ROOM    );
	act( social_table[cmd].char_no_arg,   ch, NULL, victim, TO_CHAR    );
    }
    else if ( ( victim = get_char_room( ch, arg ) ) == NULL )
    {
	OBJ_DATA *obj;

	//social to MY OBJ
	for ( obj = ch->carrying; obj != NULL; obj = obj->next_content )
    {
	if ( can_see_obj( ch, obj ) && is_name( arg, obj->name ) )
	{
	act( social_table[cmd].obj_inv_room, ch, obj, NULL, TO_ROOM );
	act( social_table[cmd].obj_inv_self, ch, obj, NULL, TO_CHAR );
	return TRUE;
	}
	}
	
	//social to ROOM OBJ
	for ( obj = ch->in_room->contents; obj != NULL; obj = obj->next_content )
    {
	if ( can_see_obj( ch, obj ) && is_name( arg, obj->name ) )
	{	
	act( social_table[cmd].obj_room_room, ch, obj, NULL, TO_ROOM );
	act( social_table[cmd].obj_room_self, ch, obj, NULL, TO_CHAR );
	return TRUE;
	}
	}
	
	//free_obj fixme
		
	send_to_char( "They aren't here.\r\n", ch );
    }
    else if ( victim == ch )
    {
	act( social_table[cmd].others_auto,   ch, NULL, victim, TO_ROOM    );
	act( social_table[cmd].char_auto,     ch, NULL, victim, TO_CHAR    );
    }
    else
    {
	act( social_table[cmd].others_found,  ch, NULL, victim, TO_NOTVICT );
	act( social_table[cmd].char_found,    ch, NULL, victim, TO_CHAR    );
	act( social_table[cmd].vict_found,    ch, NULL, victim, TO_VICT    );
    }

    return TRUE;
}



/*
 * Return true if an argument is completely numeric.
 */
bool is_number( char *arg )
{
    if ( *arg == '\0' )
	return FALSE;

    for ( ; *arg != '\0'; arg++ )
    {
	if ( !isdigit((unsigned char)*arg) )
	    return FALSE;
    }

    return TRUE;
}



/*
 * Given a string like 14.foo, return 14 and 'foo'
 */

int number_argument( char *argument, char *arg )
{
    char *pdot;
    int number;

    for ( pdot = argument; *pdot != '\0'; pdot++ )
    {
	if ( *pdot == '.' )
	{
	    *pdot = '\0';
	    number = atoi( argument );
	    *pdot = '.';
	    strcpy( arg, pdot+1 );
	    return number;
	}
    }

    strcpy( arg, argument );
    return 1;
}


/*
 * Pick off one argument from a string and return the rest.
 * Understands quotes.
 */
char *one_argument( char *argument, char *arg_first )
{
    char cEnd;

    while ( isspace((unsigned char)*argument) )
	argument++;

    cEnd = ' ';
    if ( *argument == '\'' || *argument == '"' )
	cEnd = *argument++;

    while ( *argument != '\0' )
    {
	if ( *argument == cEnd )
	{
	    argument++;
	    break;
	}
	*arg_first = LOWER(*argument);
	arg_first++;
	argument++;
    }
    *arg_first = '\0';

    while ( isspace((unsigned char)*argument) )
	argument++;

    return argument;
}


// this is an actual kludge that works...
char *parse( char *argument, DO_FUN *cmd )
{
	char *command[10];
	int i;
	static char buf[MSL];	
	int count = 0;
	
	command[0] = strtok( argument, " " );

	for( i = 1; i < 10; i++ )
	{
		command[i] = strtok( NULL, " " );
	}
	
	for( i = 0; i < 10; i++ )
	{
		if( command[i] != NULL )
		{		
			if( ( !str_cmp( command[i], "in" ) || !str_cmp( command[i], "on" ) ) && cmd != do_look )
				command[i] = NULL;
			
			else if( !str_cmp( command[i], "the" ) )
				command[i] = NULL;
			
			else if( !str_cmp( command[i], "a" ) )
				command[i] = NULL;
				
			else if( !str_cmp( command[i], "from" ) )
				command[i] = NULL;
			
			else if( !str_cmp( command[i], "my" ) )
			{
				sprintf(buf, "my*%s", command[i+1]);
				command[i+1] = str_dup(buf);
				command[i] = NULL;
			}
			
			else if( !str_cmp( command[i], "at" ) )
				command[i] = NULL;
				
			else if( !str_cmp( command[i], "to" ) )
				command[i] = NULL;
				
			else if( !str_cmp( command[i], "first" ) && i+1 < 11 && command[i+1] != NULL )
			{
				sprintf(buf, "1.%s", command[i+1]);
				command[i+1] = str_dup(buf);
				command[i] = NULL;
			}
				
			else if( !str_cmp( command[i], "second" ) && i+1 < 11 && command[i+1] != NULL )
			{
				sprintf(buf, "2.%s", command[i+1]);
				command[i+1] = str_dup(buf);
				command[i] = NULL;
			}
			
			else if( !str_cmp( command[i], "other" ) && i+1 < 11 && command[i+1] != NULL )
			{
				sprintf(buf, "2.%s", command[i+1]);
				command[i+1] = str_dup(buf);
				command[i] = NULL;
			}
			
			else if( !str_cmp( command[i], "third" ) && i+1 < 11 && command[i+1] != NULL )
			{
				sprintf(buf, "3.%s", command[i+1]);
				command[i+1] = str_dup(buf);
				command[i] = NULL;
			}
			
			else if( !str_cmp( command[i], "fourth" ) && i+1 < 11 && command[i+1] != NULL )
			{
				sprintf(buf, "4.%s", command[i+1]);
				command[i+1] = str_dup(buf);
				command[i] = NULL;
			}
			
			else if( !str_cmp( command[i], "fifth" ) && i+1 < 11 && command[i+1] != NULL )
			{
				sprintf(buf, "5.%s", command[i+1]);
				command[i+1] = str_dup(buf);
				command[i] = NULL;
			}
						
			else if( !str_cmp( command[i], "sixth" ) && i+1 < 11 && command[i+1] != NULL )
			{
				sprintf(buf, "6.%s", command[i+1]);
				command[i+1] = str_dup(buf);
				command[i] = NULL;
			}
						
			else if( !str_cmp( command[i], "seventh" ) && i+1 < 11 && command[i+1] != NULL )
			{
				sprintf(buf, "7.%s", command[i+1]);
				command[i+1] = str_dup(buf);
				command[i] = NULL;
			}
						
			else if( !str_cmp( command[i], "eighth" ) && i+1 < 11 && command[i+1] != NULL )
			{
				sprintf(buf, "8.%s", command[i+1]);
				command[i+1] = str_dup(buf);
				command[i] = NULL;
			}
						
			else if( !str_cmp( command[i], "ninth" ) && i+1 < 11 && command[i+1] != NULL )
			{
				sprintf(buf, "9.%s", command[i+1]);
				command[i+1] = str_dup(buf);
				command[i] = NULL;
			}
						
			else if( !str_cmp( command[i], "tenth" ) && i+1 < 11 && command[i+1] != NULL )
			{
				sprintf(buf, "10.%s", command[i+1]);
				command[i+1] = str_dup(buf);
				command[i] = NULL;
			}
						
			else if( !str_cmp( command[i], "eleventh" ) && i+1 < 11 && command[i+1] != NULL )
			{
				sprintf(buf, "11.%s", command[i+1]);
				command[i+1] = str_dup(buf);
				command[i] = NULL;
			}
						
			else if( !str_cmp( command[i], "twelfth" ) && i+1 < 11 && command[i+1] != NULL )
			{
				sprintf(buf, "12.%s", command[i+1]);
				command[i+1] = str_dup(buf);
				command[i] = NULL;
			}
		}
	}
	
	strcpy( buf, "" );
		
	for( i = 0; i < 10; i++ )
	{
		if( command[i] != NULL )
		{
		count++;
		}
	}
	
	for( i = 0; i < 10; i++ )
	{
		if( command[i] != NULL )
		{
		strcat( buf, command[i] );
		count--;
		
		if( count > 0 )
		strcat( buf, " " );
		}
	}
	
	return buf;
}


// for testing purposes
void do_parse( CHAR_DATA *ch, char *argument )
{
	printf_to_char(ch, "%s\r\n", parse(argument,NULL)  );
}
// Things to do:

// * Fix messaging with act

// * 


#if defined(macintosh)
#include <types.h>
#else
#include <sys/types.h>
#endif
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <time.h>
#include "merc.h"



/*
 * Local functions.
 */
void	say_spell	args( ( CHAR_DATA *ch, int sn ) );



/*
 * Lookup a skill by name.
 */
int skill_lookup( const char *name )
{
    int sn;

    for ( sn = 0; sn < MAX_SKILL; sn++ )
    {
	if ( skill_table[sn].name == NULL )
	    break;
	if ( LOWER(name[0]) == LOWER(skill_table[sn].name[0])
	&&   !str_prefix( name, skill_table[sn].name ) )
	    return sn;
    }

    return -1;
}

int spell_lookup( const char *name )
{
    int sn;
	int number = atoi(name);
    for ( sn = 0; sn < MAX_SPELL; sn++ )
    {
	if ( spell_table[sn].name == NULL )
	    break;
	
		if( spell_table[sn].number == number ) 
		return sn;
	
	if ( LOWER(name[0]) == LOWER(spell_table[sn].name[0])
	&&   !str_prefix( name, spell_table[sn].name ) )
	{
	    return sn;
    }
	}

    return -1;
}



/*
 * Lookup a skill by slot number.
 * Used for object loading.
 */
int slot_lookup( int slot )
{
    extern bool fBootDb;
    int sn;

    if ( slot <= 0 )
	return -1;

    for ( sn = 0; sn < MAX_SPELL; sn++ )
    {
	if ( slot == spell_table[sn].slot )
	    return sn;
    }

	//fixme...but will i be using stupid slots?
    if ( fBootDb )
    {
	bug( "Slot_lookup: bad slot %d.", slot );
	//abort( );
		return 0;
    }

    return -1;
}

// since we're going classless, magic will have more requirements.  You must have 1 rank of appropriate lore
// and 1 rank of spell research skill to cast a spell of such a level. e.g. level 10 spell = 10 ranks of each.

bool can_cast( CHAR_DATA *ch, short sn )
{
	if( IS_NPC( ch ) )
		return TRUE; // assuming if cast is called, the mob can cast it as the reqs have already been checked.

	if( !spell_table[sn].spell_fun || spell_table[sn].spell_fun == spell_null )
		return FALSE;

	//BLACK SORCERY IS EVIL, white magic is good, PSIONICS/GREY MAGIC LATER???
	if( spell_table[sn].prof == CLASS_MAGE || spell_table[sn].prof == CLASS_SORCERER )
	{
	if( ch->pcdata->learned[gsn_spell_research] >= spell_table[sn].level && ch->pcdata->learned[gsn_elemental_lore] >= spell_table[sn].level )
		return TRUE;
	}
	else if( spell_table[sn].prof == CLASS_HEALER || spell_table[sn].prof == CLASS_PRIEST )
	{
	if( ch->pcdata->learned[gsn_spell_research] >= spell_table[sn].level && ch->pcdata->learned[gsn_spiritual_lore] >= spell_table[sn].level )
		return TRUE;
	}
	
	return FALSE;
}

// boost duration of stackable spell
void boost_duration( CHAR_DATA *ch, int sn, int amt )
{
    AFFECT_DATA *paf;

    for ( paf = ch->affected; paf != NULL; paf = paf->next )
    {
	if ( paf->type == sn )
	    paf->duration += amt;
    }
}




void do_release( CHAR_DATA *ch, char *argument )
{
	if( !ch->prepared_spell || ch->prepared_spell < 0)
	{
		send_to_char( "But you have no prepared spell to release!\r\n", ch);
		ch->prepared_spell = 0;
		return;
	}
	else
	{
		printf_to_char( ch, "You release your spell of %s.\r\n", spell_table[ch->prepared_spell].name);
		ch->prepared_spell = 0;
		return;
    }

}

void do_finishprep(CHAR_DATA *ch, char *argument )
{
		send_to_char( "Your spell is ready to cast.\r\n", ch );
    	ch->prepared_spell = ch->predelay_info->number;
}

void do_finishprepnum( CHAR_DATA *ch, int sn )
{
		send_to_char( "Your spell is ready to cast.\r\n", ch );
    	ch->prepared_spell = sn;
}

void do_prepare( CHAR_DATA *ch, char *argument )
{
	int sn;
	
	//printf( "%s preparing %s\r\n", ch->name, argument);

	if ( argument[0] == '\0' )
	{
	    send_to_char( "Prepare what spell?\r\n", ch );
	    return;
	}
	
	if( IS_NPC(ch) && ch->prepared_spell > 0 )
	{
			
		//printf( "%s force cast %s\r\n", ch->name, argument);
		
		if( ch->hunting && spell_table[ch->prepared_spell].target == TAR_CHAR_OFFENSIVE )
			do_cast(ch, ch->hunting->name );
		else
			do_cast(ch, "");
		
		return;
	}


	if( ch->mana < 1 )
	{
			send_to_char( "You have no mana to draw upon!\r\n", ch );
		    return;
	}

	if( !IS_NPC(ch) && ch->prepared_spell > 0 )
	{
		    send_to_char( "You already have a spell prepared!\r\n", ch );
		    send_to_char( "You must release it to cast another.\r\n", ch );
		    return;
	}

	    if ( ( sn = spell_lookup( argument ) ) < 0
	    || ( spell_table[sn].spell_fun == spell_null ) )
	    {
		send_to_char( "There is no such spell.\r\n", ch );
		return;
	    }
		
		if( ch->body[BODY_CNS] > 1 || ch->scars[BODY_CNS] > 1)
		{
		printf_to_char( ch, "Your nerves are too much of a wreck for you to concentrate.r\n");
		return;
		}
		
		if( ch->body[BODY_HEAD] > 1 || ch->scars[BODY_HEAD] > 1)
		{
		printf_to_char( ch, "Your head is too badly injured for you to remember the spell.\r\n" );
		return;
		}
		
		if( ch->body[BODY_R_EYE] > 1 || ch->scars[BODY_L_EYE] > 1)
		{
		printf_to_char( ch, "Your right eye is too injured to complete the somatic gestures.\r\n");
		return;
		}
		
		if( ch->body[BODY_L_EYE] > 1 || ch->scars[BODY_L_EYE] > 1)
		{
		printf_to_char( ch, "Your left eye is too injured to complete the somatic gestures.\r\n");
		return;
		}
		
		if( ch->body[BODY_R_ARM] > 1 || ch->scars[BODY_R_ARM] > 1)
		{
		printf_to_char( ch, "Your right arm is too injured to complete the somatic gestures.\r\n");
		return;
		}

		if( ch->body[BODY_R_HAND] > 1 || ch->scars[BODY_R_HAND] > 1)
		{
		printf_to_char( ch, "Your right hand is too injured to complete the somatic gestures.\r\n");
		return;
		}
		
		if( ch->body[BODY_L_ARM] > 1 || ch->scars[BODY_L_ARM] > 1)
		{
		printf_to_char( ch, "Your left arm is too injured to complete the somatic gestures.\r\n");
		return;
		}

		if( ch->body[BODY_L_HAND] > 1 || ch->scars[BODY_L_HAND] > 1)
		{
		printf_to_char( ch, "Your left hand is too injured to complete the somatic gestures.\r\n");
		return;
		}
		
		if ( !IS_NPC(ch) )
		{
		if( !IS_IMMORTAL(ch) && !can_cast(ch, sn ) )
		{
		printf_to_char( ch, "You cannot yet control such power.\r\n" );
		return;
		}
		}
		
		if( !IS_NPC(ch) && ch->level < spell_table[sn].level )
	    {
		send_to_char( "You cannot yet harness such power.\r\n", ch );
		return;
	    }
		
		//fixme by spell sphere
		if( spell_table[sn].prof == CLASS_PRIEST )
		{
    	printf_to_char(ch, "You recite the words to %s...\r\n", spell_table[sn].name );
    	act( "$n begins to recite some words...", ch, NULL, NULL, TO_ROOM );
		}
		else if( spell_table[sn].prof == CLASS_HEALER )
		{
    	printf_to_char(ch, "You murmur a chant of %s...\r\n", spell_table[sn].name );
    	act( "$n begins to murmur a chant...", ch, NULL, NULL, TO_ROOM );
		}
		else if ( spell_table[sn].prof == CLASS_MAGE )
		{
    	printf_to_char(ch, "You invoke your spell of %s...\r\n", spell_table[sn].name );
    	act( "$n begins to mutter some incantations...", ch, NULL, NULL, TO_ROOM );
		}
		else if ( spell_table[sn].prof == CLASS_SORCERER )
		{
    	printf_to_char(ch, "You utter a dark invocation of %s...\r\n", spell_table[sn].name );
    	act( "$n begins to utter a dark invocation..", ch, NULL, NULL, TO_ROOM );
		}
		else
		{
    	printf_to_char(ch, "You prepare your spell of %s...\r\n", spell_table[sn].name );
    	act( "$n begins preparing a spell...", ch, NULL, NULL, TO_ROOM );
		}
		
		/* void set_predelay( CHAR_DATA *ch, int delay, DO_FUN *fnptr, char *argument,
		int number, CHAR_DATA *victim1, CHAR_DATA *victim2, OBJ_DATA *obj1,
		OBJ_DATA *obj2 ) */
		{
			short time = 10*PULSE_PER_SECOND;
			
			//fixme training vs level based?
			if( ch->level > spell_table[sn].level + 2 ) 
				time -= 5*PULSE_PER_SECOND;
			
			if( ch->level > spell_table[sn].level + 5 )
				time -= 5*PULSE_PER_SECOND;
			
			if( time > 0 )
			{
		    set_predelay( ch, time, do_finishprep, "", sn, NULL, NULL, NULL, NULL, FALSE );
			printf_to_char(ch, "(Spell will be ready in %d seconds.)\r\n", time / PULSE_PER_SECOND ) ;
			}
			else
			do_finishprepnum( ch, sn );
		}
}



/*
 * The kludgy global is for spells who want more stuff from command line.
 */
char *target_name;

// For spells that want mobile _or_ object targets.

bool targ_mob;


void do_cast( CHAR_DATA *ch, char *argument )
{
    CHAR_DATA *victim;
    OBJ_DATA *obj;
    void *vo;
    int mana;
    int sn = ch->prepared_spell;

    /*
     * Switched NPC's can cast spells, but others can't.
     */
	 
	 // nah, they can
	 
    //if ( IS_NPC(ch) && ch->desc == NULL )
	//return;

	if( !sn || sn < 0)
	{
		send_to_char("You don't have a spell prepared!\r\n", ch );
		return;
	}

	if( spell_table[sn].spell_fun == spell_null )
	{
			send_to_char("That's not a spell.\r\n", ch );
			return;
	}

	if( !IS_NPC(ch) && !IS_IMMORTAL(ch) && !can_cast( ch, sn ) )
	{
	printf_to_char( ch, "You cannot cast %s.\r\n",
	spell_table[sn].name );
	return;
	}
    mana = spell_table[sn].level;

    /*
     * Locate targets.
     */
    victim	= NULL;
    obj		= NULL;
    vo		= NULL;

    target_name = str_dup( argument );

    switch ( spell_table[sn].target )
    {
    default:
	bug( "Do_cast: bad target for sn %d.", sn );
	return;

    case TAR_IGNORE:
	break;

    case TAR_CHAR_OFFENSIVE:

	    if ( ( victim = get_char_room( ch, argument ) ) == NULL )
	    {
		send_to_char( "Cast at whom?\r\n", ch );
		return;
	    }
		
		if ( is_safe( ch, victim ) && sn != gsn_animate_corpse )
	    {
		return;
	    }

	vo = (void *) victim;

	if( victim != ch )
	{
	    printf_to_char( ch, "You gesture at %s.\r\n", PERS( victim, ch ) );
    	act( "$n gestures at $N.", ch, NULL, victim, TO_ROOM );
	}
	else
	{
	    printf_to_char( ch, "You gesture.\r\n" );
    	act( "$n gestures.", ch, NULL, NULL, TO_ROOM );
	}
	
	break;

    case TAR_CHAR_DEFENSIVE:
	case TAR_CHAR_HEALING:
	if ( argument[0] == '\0' )
	{
	    victim = ch;
	}
	else
	{
	    if ( ( victim = get_char_room( ch, argument ) ) == NULL )
	    {
		send_to_char( "Cast at whom?\r\n", ch );
		return;
	    }
		
		if ( IS_SET( victim->affected_by, AFF_DEAD ) && sn != gsn_resurrect
			&& spell_table[sn].target != TAR_CHAR_HEALING		)
		{
		printf_to_char(ch, "%s is dead, though.\r\n", HE_SHE(victim) );
		return;
		}
	}

	vo = (void *) victim;
	
	if( victim == NULL )
		return;

	if( victim != ch )
	{
	    printf_to_char( ch, "You gesture at %s.\r\n", PERS( victim, ch ) );
    	act( "$n gestures at $N.", ch, NULL, victim, TO_NOTVICT );
		act( "$n gestures at you.", ch, NULL, victim, TO_VICT );
	}
	else
	{
		printf_to_char( ch, "You gesture.\r\n"  );
    	act( "$n gestures.", ch, NULL, NULL, TO_ROOM );
	}
	break;

    case TAR_CHAR_SELF:
	if ( argument[0] != '\0' && !is_name( argument, ch->name ) )
	{
	    send_to_char( "You cannot cast this spell on another.\r\n", ch );
	    return;
	}

	vo = (void *) ch;

	    printf_to_char( ch, "You gesture.\r\n"  );
    	act( "$n gestures.", ch, NULL, NULL, TO_ROOM );
	break;

    case TAR_OBJ_INV:
	if ( argument[0] == '\0' )
	{
	    send_to_char( "What should the spell be cast upon?\r\n", ch );
	    return;
	}

	if ( ( obj = get_obj_carry( ch, argument ) ) == NULL )
	{
	    send_to_char( "You are not carrying that.\r\n", ch );
	    return;
	}

	vo = (void *) obj;

	    printf_to_char( ch, "You gesture at %s.\r\n", obj->short_descr  );
    	act( "$n gestures at $p.", ch, obj, NULL, TO_ROOM );
	break;

	
	case TAR_EITHER:
	if ( ( victim = get_char_room( ch, argument ) ) != NULL )
	{
	    targ_mob = TRUE;
	    vo = (void *) victim;
	}
	else if ( ( obj = get_obj_carry( ch, argument ) ) != NULL )
	{
	    targ_mob = FALSE;
	    vo = (void *) obj;
	}
	else
	{
	    send_to_char("No such player or object here.\n\r", ch );
	    return;
	}
	break;
    }

    if ( !IS_NPC(ch) && ch->mana < mana )
    {
	send_to_char( "You don't have enough mana.\r\n", ch );
	return;
    }

	/*
    if ( str_cmp( spell_table[sn].name, "ventriloquate" ) )
	say_spell( ch, sn );
	*/
	
	if( !vo )
	{
 		printf_to_char( ch, "You gesture.\r\n"  );
    	act( "$n gestures.", ch, NULL, NULL, TO_ROOM );
	}

	ch->mana -= mana;
	
	//fixme greaves, helmets, etc should add to hindrance
	{
		short hind = 0;
		short roll = number_range(1,100);
		OBJ_DATA *armor = get_eq_char( ch, EQUIP_CHEST );
		
		if( armor != NULL )
		{
		if( !IS_NPC(ch) )
		{
		if( armor->value[3] > 0 && armor->value[3] < MAX_ARMOR )
		{
		hind = armor_table[armor->value[3]].HindranceMax;
		}
		
		hind -= ch->pcdata->learned[gsn_armor_use] / 2;
		
		if( hind < armor_table[armor->value[3]].Hindrance)
			hind = armor_table[armor->value[3]].Hindrance;
		
		}
		else
		hind = armor_table[armor->value[3]].Hindrance;		
		}
		// no armr, hind = 0
		
		if( spell_table[sn].target == TAR_CHAR_OFFENSIVE && roll == 1 )
		{
		printf_to_char( ch, "d100 = 1 FUMBLE!  Your magic fails.\r\n"  );
    	act( "d100 = 1 FUMBLE!  $n's magic fails.", ch, NULL, NULL, TO_ROOM );
		ch->prepared_spell = 0;
		add_roundtime( ch, 3 );
        show_roundtime(ch, 3 );
		return;
		}
		else if( roll <= hind && armor != NULL )
		{
		printf_to_char( ch, "Your armor prevents the spell from working.\r\n[Spell hindrance of %s is %d percent.]\r\n",
		armor->short_descr, hind );
    	act( "$n's armor prevents the spell from working.", ch, NULL, NULL, TO_ROOM );
		ch->prepared_spell = 0;
		add_roundtime( ch, 3 );
        show_roundtime(ch, 3 );
		return;
		}
	}
	
	(*spell_table[sn].spell_fun) ( sn, ch->level, ch, vo );
    ch->prepared_spell = 0;

//fixme cast rt?? all RT is currently hard.

        add_roundtime( ch, 3 );
		
        show_roundtime(ch, 3 );

		if( ch->stance == 100 )
		{
			ch->stance = 80;
			printf_to_char( ch, "(Forcing your stance to guarded.)\r\n" );
		}
		
    return;
}



/*
 * Cast spells at targets using a magical object.
 */
void obj_cast_spell( int sn, int level, CHAR_DATA *ch, CHAR_DATA *victim, OBJ_DATA *obj )
{
    void *vo;

    if ( sn <= 0 )
	return;

    if ( sn >= MAX_SPELL || spell_table[sn].spell_fun == 0 )
    {
	bug( "Obj_cast_spell: bad sn %d.", sn );
	return;
    }

    switch ( spell_table[sn].target )
    {
    default:
	bug( "Obj_cast_spell: bad target for sn %d.", sn );
	return;

    case TAR_IGNORE:
	vo = NULL;
	break;

    case TAR_CHAR_OFFENSIVE:
	if ( victim == NULL )
	{
	    send_to_char( "You require a target for this spell.\r\n", ch );
	    return;
	}
	vo = (void *) victim;
	break;

    case TAR_CHAR_DEFENSIVE:
	case TAR_CHAR_HEALING:
	if ( victim == NULL )
	    victim = ch;
	vo = (void *) victim;
	break;

    case TAR_CHAR_SELF:
	vo = (void *) ch;
	break;

    case TAR_OBJ_INV:
	if ( obj == NULL )
	{
	    send_to_char( "You require an item as a target for this spell.\r\n", ch );
	    return;
	}
	vo = (void *) obj;
	break;
    }

    target_name = "";
    (*spell_table[sn].spell_fun) ( sn, level, ch, vo );

    return;
}

/*
0	"surge of electricity", DAMAGE_TYPE_SHOCK, 150, 133, 111, 122, 128
1	"stream of water", 	DAMAGE_TYPE_WATER, 455, 345, 283, 242, 173
2	"stream of acid", 	DAMAGE_TYPE_ACID, 525, 383, 314, 350, 197
3	"stream of fire", 	DAMAGE_TYPE_FIRE, 667, 455, 345, 323, 303
4	"ball of ice",		DAMAGE_TYPE_COLD, 445, 350, 245, 217, 208
5	"ball of fire", 	DAMAGE_TYPE_FIRE, 400, 333, 270, 256, 244
6	"bolt of lightning", DAMAGE_TYPE_SHOCK, 750, 555, 433, 415, 433
7	"massive boulder", DAMAGE_TYPE_CRUSH, 710, 520, 460, 435, 440
8	"disintegrating ray", DAMAGE_TYPE_DISINTEGRATE, 525, 383, 314, 350, 197
9	"hissing cloud of steam", DAMAGE_TYPE_FIRE, 600, 550, 385, 333, 267
*/


void spell_fire_bolt( int sn, int level, CHAR_DATA *ch, void *vo )
{
    CHAR_DATA *victim = (CHAR_DATA *) vo;
	short dam;
	
	
	if( ch != victim )
	{
	act( "$n hurls a sizzling stream of fire at $N!", ch, NULL, victim, TO_NOTVICT );
    act( "You hurl a sizzling stream of fire at $N!", ch, NULL, victim, TO_CHAR );
    act( "$n hurls a sizzling stream of fire at you!", ch, NULL, victim, TO_VICT );	
	}
	else
	{
    act( "You hurl a sizzling stream of fire at yourself!", ch, NULL, victim, TO_CHAR );
    act( "$n hurls a sizzling stream of fire at $mself!", ch, NULL, victim, TO_ROOM );		
	}
	
    dam = bolt_check( ch, victim, sn );
	
	
	if( dam != 0 )
		damage( ch, victim, dam, 2003 );

    return;
}

void spell_fireball( int sn, int level, CHAR_DATA *ch, void *vo )
{
    CHAR_DATA *victim = (CHAR_DATA *) vo;
	short dam;
	
	if( ch != victim )
	{
	act( "$n hurls a roaring ball of fire at $N!", ch, NULL, victim, TO_NOTVICT );
    act( "You hurl a roaring ball of fire at $N!", ch, NULL, victim, TO_CHAR );
    act( "$n hurls a roaring ball of fire at you!", ch, NULL, victim, TO_VICT );	
	}
	else
	{
    act( "You hurl a roaring ball of fire at yourself!", ch, NULL, victim, TO_CHAR );
    act( "$n hurls a roaring ballof fire at $mself!", ch, NULL, victim, TO_ROOM );		
	}
	
    dam = bolt_check( ch, victim, sn );
	
	
	if( dam != 0 )
		damage( ch, victim, dam, 2005 );

    return;
}

void spell_shock_bolt( int sn, int level, CHAR_DATA *ch, void *vo )
{
    CHAR_DATA *victim = (CHAR_DATA *) vo;
	short dam;
	
	if( ch != victim )
	{
	act( "$n aims an arc of streaming sparks at $N!", ch, NULL, victim, TO_NOTVICT );
    act( "You aim an arc of streaming sparks at $N!", ch, NULL, victim, TO_CHAR );
    act( "$n aim an arc of streaming sparks at you!", ch, NULL, victim, TO_VICT );	
	}
	else
	{
    act( "You aim an arc of streaming sparks at yourself!", ch, NULL, victim, TO_CHAR );
    act( "$n aim an arc of streaming sparks $mself!", ch, NULL, victim, TO_ROOM );		
	}	
	
    dam = bolt_check( ch, victim, sn );
	
	
	if( dam != 0 )
		damage( ch, victim, dam, 2000 );

    return;
}

void spell_lightning_bolt( int sn, int level, CHAR_DATA *ch, void *vo )
{
    CHAR_DATA *victim = (CHAR_DATA *) vo;
	short dam;
	
	if( ch != victim )
	{
	act( "$n hurls a crackling bolt of lightning at $N!", ch, NULL, victim, TO_NOTVICT );
    act( "You hurl a crackling bolt of lightning at $N!", ch, NULL, victim, TO_CHAR );
    act( "$n hurls a crackling bolt of lightning at you!", ch, NULL, victim, TO_VICT );	
	}
	else
	{
    act( "You hurl a crackling bolt of lightning at yourself!", ch, NULL, victim, TO_CHAR );
    act( "$n hurls a crackling bolt of lightning at $mself!", ch, NULL, victim, TO_ROOM );		
	}	
	
    dam = bolt_check( ch, victim, sn );
	
	
	if( dam != 0 )
		damage( ch, victim, dam, 2006 );

    return;
}

void spell_water_bolt( int sn, int level, CHAR_DATA *ch, void *vo )
{
    CHAR_DATA *victim = (CHAR_DATA *) vo;
	short dam;
	
	if( ch != victim )
	{
	act( "$n hurls a rushing torrent of water at $N!", ch, NULL, victim, TO_NOTVICT );
    act( "You hurl a rushing torrent of water at $N!", ch, NULL, victim, TO_CHAR );
    act( "$n hurls a rushing torrent of water at you!", ch, NULL, victim, TO_VICT );	
	}
	else
	{
    act( "You hurl a rushing torrent of water at yourself!", ch, NULL, victim, TO_CHAR );
    act( "$n hurls a rushing torrent of water at $mself!", ch, NULL, victim, TO_ROOM );		
	}	
	
    dam = bolt_check( ch, victim, sn );
	
	
	if( dam != 0 )
		damage( ch, victim, dam, 2001 );

    return;
}

void spell_steam( int sn, int level, CHAR_DATA *ch, void *vo )
{
    CHAR_DATA *victim = (CHAR_DATA *) vo;
	short dam;
	
	if( ch != victim )
	{
	act( "$n sends a boiling cloud of steam at $N!", ch, NULL, victim, TO_NOTVICT );
    act( "You send a boiling cloud of steam at $N!", ch, NULL, victim, TO_CHAR );
    act( "$n sends a boiling cloud of steam at you!", ch, NULL, victim, TO_VICT );	
	}
	else
	{
    act( "You send a boiling cloud of steam at yourself!", ch, NULL, victim, TO_CHAR );
    act( "$n sends a boiling cloud of steam at $mself!", ch, NULL, victim, TO_ROOM );		
	}	
	
    dam = bolt_check( ch, victim, sn );
	
	
	if( dam != 0 )
		damage( ch, victim, dam, 2009 );

    return;
}


void spell_holy_bolt( int sn, int level, CHAR_DATA *ch, void *vo )
{
    CHAR_DATA *victim = (CHAR_DATA *) vo;
	short dam;
	
	if( ch != victim )
	{
	act( "$n hurls a stream of pure water at $N!", ch, NULL, victim, TO_NOTVICT );
    act( "You hurl a stream of pure water at $N!", ch, NULL, victim, TO_CHAR );
    act( "$n hurls a stream of pure water at you!", ch, NULL, victim, TO_VICT );	
	}
	else
	{
    act( "You hurl a stream of pure water at yourself!", ch, NULL, victim, TO_CHAR );
    act( "$n hurls a stream of pure water at $mself!", ch, NULL, victim, TO_ROOM );		
	}

    dam = bolt_check( ch, victim, sn );
	
	if( dam != 0 )
	{
	if( IS_SET(victim->affected_by, AFF_UNDEAD ) )
		damage( ch, victim, dam, 2002 ); // acid
	else
		damage( ch, victim, dam, 2001 ); // water
	}
	
    return;
}


void spell_stone( int sn, int level, CHAR_DATA *ch, void *vo )
{
    CHAR_DATA *victim = (CHAR_DATA *) vo;
	short dam;
	
	if( ch != victim )
	{
	act( "$n hurls a huge rocky boulder at $N!", ch, NULL, victim, TO_NOTVICT );
    act( "You hurl a huge rocky boulder at $N!", ch, NULL, victim, TO_CHAR );
    act( "$n hurls a huge rocky boulder at you!", ch, NULL, victim, TO_VICT );	
	}
	else
	{
    act( "You hurl a huge rocky boulder at yourself!", ch, NULL, victim, TO_CHAR );
    act( "$n hurls a huge rocky boulder at $mself!", ch, NULL, victim, TO_ROOM );		
	}	
	
    dam = bolt_check( ch, victim, sn );
	
	
	if( dam != 0 )
		damage( ch, victim, dam, 2007 );

    return;
}


void spell_harm( int sn, int level, CHAR_DATA *ch, void *vo )
{
    CHAR_DATA *victim = (CHAR_DATA *) vo;
    short dam = warding_check( ch, victim );
	
	if( dam != 0 )
	{
		if( !IS_SET( victim->affected_by, AFF_UNDEAD ) )
		damage( ch, victim, dam * 455 / 1000, sn );
		else
		damage( ch, victim, dam * 211 / 1000, sn ); 
	}
    return;
}

void spell_sleep( int sn, int level, CHAR_DATA *ch, void *vo )
{
    AFFECT_DATA af;
	CHAR_DATA *victim = (CHAR_DATA *) vo;
    short dam = warding_check( ch, victim );
	
	if ( !IS_AWAKE(victim) || IS_AFFECTED(victim, AFF_SLEEP) )
	{
		printf_to_char( ch, "The magic has no effect as %s is already sleep.\r\n", HE_SHE(victim) );
		return;
	}
	
	if( dam != 0 )
	{
			
    af.type      = sn;
    af.duration  = dam;
    af.location  = APPLY_NONE;
    af.modifier  = 0;
    af.bitvector = AFF_SLEEP;
	
	send_to_char( "You suddenly feel very tired, slump over, and go to sleep.\r\n", victim );
	act( "$n immediately slumps over and goes to sleep.", victim, NULL, NULL, TO_ROOM );
	
    affect_join( victim, &af );
	victim->position = POS_PRONE;
	
	}
	
    return;
}


void spell_hold_undead( int sn, int level, CHAR_DATA *ch, void *vo )
{
    CHAR_DATA *victim = (CHAR_DATA *) vo;
    short margin = warding_check( ch, victim );
    
    if( !IS_SET( victim->affected_by, AFF_UNDEAD ) )
	{
		act( "A glowing white mist surrounds $N but quickly fades.", ch, NULL, victim, TO_NOTVICT );
		act( "A glowing white mist surrounds $N but quickly fades.", ch, NULL, victim, TO_CHAR );
		act( "A glowing white mist surrounds you but quickly fades.", ch, NULL, victim, TO_VICT );

	margin = 0;
	}
	
    if( margin != 0 )
	{
	    AFFECT_DATA af;
	
		if ( IS_AFFECTED(victim, AFF_HOLD) || is_affected(victim, gsn_hold_undead) )
		{
			printf_to_char( ch, "The magic has no effect as %s is already held.\r\n", HE_SHE(victim) );
			return;
		}
				
		af.type      = sn;
		af.duration  = margin+50;
		af.location  = APPLY_NONE;
		af.modifier  = 0;
		af.bitvector = AFF_HOLD;
	
		send_to_char( "You are held fast!\r\n", victim );
		act( "$n is held fast!", victim, NULL, NULL, TO_ROOM );
		affect_join( victim, &af );
	}
	
    return;
}

void spell_destroy_undead( int sn, int level, CHAR_DATA *ch, void *vo )
{
    CHAR_DATA *victim = (CHAR_DATA *) vo;
    short dam = warding_check( ch, victim );
    
    if( !IS_SET( victim->affected_by, AFF_UNDEAD ) )
	{
		act( "A glowing white mist surrounds $N but quickly fades.", ch, NULL, victim, TO_NOTVICT );
		act( "A glowing white mist surrounds $N but quickly fades.", ch, NULL, victim, TO_CHAR );
		act( "A glowing white mist surrounds you but quickly fades.", ch, NULL, victim, TO_VICT );

	dam = 0;
	}
	
    if( dam != 0 )
	damage( ch, victim, dam * 600 / 1000, sn );
	
    return;
}

void spell_jolt( int sn, int level, CHAR_DATA *ch, void *vo )
{
    CHAR_DATA *victim = (CHAR_DATA *) vo;
	
	if( victim->stun > 0 )
	{
		act( "Nothing happens.", ch, NULL, victim, TO_NOTVICT );
		act( "Nothing happens.", ch, NULL, victim, TO_CHAR );
		act( "Nothing happens.", ch, NULL, victim, TO_VICT );
		return;		
	}
	else
	{
    short dam = warding_check( ch, victim );
	
	if( dam != 0 )
	{
	if( dam > 100 )
		dam = 100;
	
	dam /= 10;
	
	if( dam < 1 )
	dam = 1;
	} // 10 round stun max
	
	if( dam != 0 )
	{
		add_stun( victim, dam );
		
		act( "$N is stunned!", ch, NULL, victim, TO_NOTVICT );
		act( "$N is stunned!", ch, NULL, victim, TO_CHAR );
		act( "You are stunned!", ch, NULL, victim, TO_VICT );	
	}
	else
	{
		act( "Nothing happens.", ch, NULL, victim, TO_NOTVICT );
		act( "Nothing happens.", ch, NULL, victim, TO_CHAR );
		act( "Nothing happens.", ch, NULL, victim, TO_VICT );	
	}
	}
		
    return;
}


void spell_death_ray( int sn, int level, CHAR_DATA *ch, void *vo )
{
    CHAR_DATA *victim = (CHAR_DATA *) vo;
	short dam;
	
	act( "$n channels a beam of grey energy at $N!", ch, NULL, victim, TO_NOTVICT );
    act( "You channel beam of grey energy at $N!", ch, NULL, victim, TO_CHAR );
    act( "$n channels a beam of grey energy at you!", ch, NULL, victim, TO_VICT );	
	
    dam = bolt_check( ch, victim, sn );
	
	
	if( dam != 0 )
		damage( ch, victim, dam, 2008 );

    return;
}

void spell_unstun( int sn, int level, CHAR_DATA *ch, void *vo )
{
    CHAR_DATA *victim = (CHAR_DATA *) vo;

		victim->stun = 0;
		
		act( "$N is no longer stunned.", ch, NULL, victim, TO_NOTVICT );
		act( "$N is no longer stunned.", ch, NULL, victim, TO_CHAR );
		act( "You are no longer stunned.", ch, NULL, victim, TO_VICT );	
		
    return;
}

void spell_cone_of_lightning( int sn, int level, CHAR_DATA *ch, void *vo )
{
	CHAR_DATA *vch;
    CHAR_DATA *vch_next;
	int dam;
	
    for ( vch = char_list; vch != NULL; vch = vch_next )
    {
	vch_next	= vch->next;
	if ( vch->in_room == NULL )
	    continue;
	if ( vch->in_room == ch->in_room )
	{
		if ( IS_DEAD(vch) )
			continue;
		
	    if ( vch != ch && ( IS_NPC(ch) ? !IS_NPC(vch) : IS_NPC(vch) ) )
		{
			act( "$n hurls a powerful bolt of lightning at $N!", ch, NULL, vch, TO_NOTVICT );
			act( "You hurl a powerful bolt of lightning at $N!", ch, NULL, vch, TO_CHAR );
			act( "$n hurls a powerful bolt of lightning at you!", ch, NULL, vch, TO_VICT );	
	
			dam = bolt_check( ch, vch, sn );
	
			if( dam != 0 )
			damage( ch, vch, dam, 2006 );
		}
	    continue;
	}

    }

    return;
}

void spell_elemental_defense_i( int sn, int level, CHAR_DATA *ch, void *vo )
{
	CHAR_DATA *victim = (CHAR_DATA *) vo;
    AFFECT_DATA af;

    if ( is_affected( victim, sn ) )
	{
	boost_duration( victim, sn, level );
	}
	else
	{
    af.type      = sn;
    af.duration  = level;
    af.location  = APPLY_DS;
    af.modifier  = 5;
    af.bitvector = 0;
    affect_to_char( victim, &af );
	}
	
	if( ch == victim )
	{
	act( "$n glows with a faint silvery light.", ch, NULL, NULL, TO_ROOM );
	act( "You glow with a faint silvery light.", ch, NULL, NULL, TO_CHAR );	
	}
	else
	{
	act( "$N glows with a faint silvery light.", ch, NULL, victim, TO_NOTVICT );
	act( "$N glows with a faint silvery light.", ch, NULL, victim, TO_CHAR );
	act( "You glow with a faint silvery light.", ch, NULL, victim, TO_VICT );	
	}
	
    return;
}

void spell_elemental_defense_ii( int sn, int level, CHAR_DATA *ch, void *vo )
{
	CHAR_DATA *victim = (CHAR_DATA *) vo;
    AFFECT_DATA af;

    if ( is_affected( victim, sn ) )
	{
	boost_duration( victim, sn, level );
	}
	else
	{
    af.type      = sn;
    af.duration  = level;
    af.location  = APPLY_DS;
    af.modifier  = 10;
    af.bitvector = 0;
    affect_to_char( victim, &af );
	}
	//ch, NULL, victim,
	if( ch == victim )
	{
	act( "$n glows with a bright silvery light.", ch, NULL, NULL, TO_ROOM );
	act( "You glow with a bright silvery light.", ch, NULL, NULL, TO_CHAR );	
	}
	else
	{
	act( "$N glows with a bright silvery light.", ch, NULL, victim, TO_NOTVICT );
	act( "$N glows with a bright silvery light.", ch, NULL, victim,TO_CHAR );
	act( "You glow with a bright silvery light.", ch, NULL, victim,TO_VICT );	
	}
	
    return;
}

void spell_elemental_defense_iii( int sn, int level, CHAR_DATA *ch, void *vo )
{
	CHAR_DATA *victim = (CHAR_DATA *) vo;
    AFFECT_DATA af;

    if ( is_affected( victim, sn ) )
	{
	boost_duration( victim, sn, level );
	}
	else
	{
    af.type      = sn;
    af.duration  = level;
    af.location  = APPLY_DS;
    af.modifier  = 15;
    af.bitvector = 0;
    affect_to_char( victim, &af );
	}
	
	if( ch == victim )
	{
	act( "$n glows with a brilliant silvery light.", ch, NULL, NULL, TO_ROOM );
	act( "You glow with a brilliant silvery light.", ch, NULL, NULL, TO_CHAR );	
	}
	else
	{
	act( "$N glows with a brilliant silvery light.", ch, NULL, victim, TO_NOTVICT );
	act( "$N glows with a brilliant silvery light.", ch, NULL, victim, TO_CHAR );
	act( "You glow with a brilliant silvery light.", ch, NULL, victim, TO_VICT );	
	}
	
    return;
}

void spell_spirit_warding_i( int sn, int level, CHAR_DATA *ch, void *vo )
{
	CHAR_DATA *victim = (CHAR_DATA *) vo;
    AFFECT_DATA af;

    if ( is_affected( victim, sn ) )
	{
	boost_duration( victim, sn, level );
	}
	else
	{
    af.type      = sn;
    af.duration  = level;
    af.location  = APPLY_TD;
    af.modifier  = 10;
    af.bitvector = 0;
    affect_to_char( victim, &af );
	}
	
	if( ch == victim )
	{
	act( "$n start shining with a light green aura.", ch, NULL, NULL, TO_ROOM );
	act( "You start to shine with a light green aura.", ch, NULL, NULL, TO_CHAR );	
	}
	else
	{
	act( "$N starts shining with a light green aura.", ch, NULL, victim, TO_NOTVICT );
	act( "$N starts shining with a light green aura.", ch, NULL, victim, TO_CHAR );
	act( "You start to shine with a light green aura.", ch, NULL, victim, TO_VICT );	
	}
	
    return;
}

void spell_spirit_warding_ii( int sn, int level, CHAR_DATA *ch, void *vo )
{
	CHAR_DATA *victim = (CHAR_DATA *) vo;
    AFFECT_DATA af;

    if ( is_affected( victim, sn ) )
	{
	boost_duration( victim, sn, level );
	}
	else
	{
    af.type      = sn;
    af.duration  = level;
    af.location  = APPLY_TD;
    af.modifier  = 25;
    af.bitvector = 0;
    affect_to_char( victim, &af );
	}
	
	if( ch == victim )
	{
	act( "$n begin to glow with a deep green aura.", ch, NULL, NULL, TO_ROOM );
	act( "You begin to glow with a deep green aura.", ch, NULL, NULL, TO_CHAR );	
	}
	else
	{
	act( "$N begins to glow with a deep green aura.", ch, NULL, victim, TO_NOTVICT );
	act( "$N begins to glow with a deep green aura.", ch, NULL, victim, TO_CHAR );
	act( "You begin to glow with a deep green aura.", ch, NULL, victim, TO_VICT );	
	}
	
    return;
}

void spell_spirit_defense( int sn, int level, CHAR_DATA *ch, void *vo )
{
	CHAR_DATA *victim = (CHAR_DATA *) vo;
    AFFECT_DATA af;

	// does not stack
    if ( is_affected( victim, sn ) )
	{
	return;
	}
	else
	{
    af.type      = sn;
    af.duration  = level;
    af.location  = APPLY_DS;
    af.modifier  = 10;
    af.bitvector = 0;
    affect_to_char( victim, &af );
	}
	
	if( ch == victim )
	{
	act( "$n suddenly appears more powerful.", ch, NULL, NULL, TO_ROOM );
	act( "You suddenly appear more powerful.", ch, NULL, NULL, TO_CHAR );	
	}
	else
	{
	act( "$N suddenly appears more powerful.", ch, NULL, victim, TO_NOTVICT );
	act( "$N suddenly appears more powerful.", ch, NULL, victim, TO_CHAR );
	act( "You suddenly appear more powerful.", ch, NULL, victim, TO_VICT );	
	}
	
    return;
}

void spell_spirit_shield( int sn, int level, CHAR_DATA *ch, void *vo )
{
	CHAR_DATA *victim = (CHAR_DATA *) vo;
    AFFECT_DATA af;

	// does not stack
    if ( is_affected( victim, sn ) )
	{
	return;
	}
	else
	{
    af.type      = sn;
    af.duration  = level;
    af.location  = APPLY_DS;
    af.modifier  = 10;
    af.bitvector = 0;
    affect_to_char( victim, &af );
	}
	
	if( ch == victim )
	{
	act( "$n is surrounded with a faint effervescence.", ch, NULL, NULL, TO_ROOM );
	act( "You are surrounded with a faint effervescence.", ch, NULL, NULL, TO_CHAR );	
	}
	else
	{
	act( "$N is surrounded with a faint effervescence.", ch, NULL, victim, TO_NOTVICT );
	act( "$N is surrounded with a faint effervescence.", ch, NULL, victim, TO_CHAR );
	act( "You are surrounded with a faint effervescence.", ch, NULL, victim, TO_VICT );	
	}
	
    return;
}

void spell_spell_shield( int sn, int level, CHAR_DATA *ch, void *vo )
{
	CHAR_DATA *victim = (CHAR_DATA *) vo;
    AFFECT_DATA af;

	// does not stack
    if ( is_affected( victim, sn ) )
	{
	return;
	}
	else
	{
    af.type      = sn;
    af.duration  = level;
    af.location  = APPLY_TD;
    af.modifier  = 20;
    af.bitvector = 0;
    affect_to_char( victim, &af );
	}
	
	if( ch == victim )
	{
	act( "$n glows with an opalescent aura.", ch, NULL, NULL, TO_ROOM );
	act( "You glow with an opalescent aura.", ch, NULL, NULL, TO_CHAR );	
	}
	else
	{
	act( "$N glows with an opalescent aura.", ch, NULL, victim, TO_NOTVICT );
	act( "$N glows with an opalescent aura.", ch, NULL, victim, TO_CHAR );
	act( "You glow with an opalescent aura..", ch, NULL, victim, TO_VICT );	
	}
	
    return;
}

void spell_lesser_shroud( int sn, int level, CHAR_DATA *ch, void *vo )
{
	CHAR_DATA *victim = (CHAR_DATA *) vo;
    AFFECT_DATA af;

    if ( is_affected( victim, sn ) )
	{
	boost_duration( victim, sn, level );
	}
	else
	{
    af.type      = sn;
    af.duration  = level;
    af.location  = APPLY_DS;
    af.modifier  = 15;
    af.bitvector = 0;
    affect_to_char( victim, &af );
	
	af.type      = sn;
    af.duration  = level;
    af.location  = APPLY_TD;
    af.modifier  = 20;
    af.bitvector = 0;
    affect_to_char( victim, &af );
	}
	
	if( ch == victim )
	{
	act( "$n suddenly appears very powerful.", ch, NULL, NULL, TO_ROOM );
	act( "You suddenly appear very powerful.", ch, NULL, NULL, TO_CHAR );	
	}
	else
	{
	act( "$N suddenly appears very powerful.", ch, NULL, victim, TO_NOTVICT );
	act( "$N suddenly appears very powerful.", ch, NULL, victim, TO_CHAR );
	act( "You suddenly appear very powerful.", ch, NULL, victim, TO_VICT );	
	}
	
    return;
}


void spell_spirit_barrier( int sn, int level, CHAR_DATA *ch, void *vo )
{
	CHAR_DATA *victim = (CHAR_DATA *) vo;
    AFFECT_DATA af;

    if ( is_affected( victim, sn ) )
	{
	boost_duration( victim, sn, level );
	}
	else
	{
    af.type      = sn;
    af.duration  = level;
    af.location  = APPLY_DS;
    af.modifier  = 20;
    af.bitvector = 0;
    affect_to_char( victim, &af );
	
	af.type      = sn;
    af.duration  = level;
    af.location  = APPLY_AS;
    af.modifier  = -20;
    af.bitvector = 0;
    affect_to_char( victim, &af );
	}
	
	if( ch == victim )
	{
	act( "$n is surrounded by a churning barrier of air.", ch, NULL, NULL, TO_ROOM );
	act( "You are surrounded by a churning barrier of air.", ch, NULL, NULL, TO_CHAR );	
	}
	else
	{
	act( "$N is surrounded by a churning barrier of air.", ch, NULL, victim, TO_NOTVICT );
	act( "$N is surrounded by a churning barrier of air.", ch, NULL, victim, TO_CHAR );
	act( "You are surrounded by a churning barrier of air.", ch, NULL, victim, TO_VICT );	
	}
	
	
    return;
}

/*
void spell_spirit_barrier( int sn, int level, CHAR_DATA *ch, void *vo )
{
	CHAR_DATA *victim = (CHAR_DATA *) vo;
    AFFECT_DATA af, af2;

    if ( is_affected( victim, sn ) )
	{
	boost_duration( victim, sn, level );
	}
	else
	{	
	af2.type      = sn;
    af2.duration  = level;
	
    af2.location  = APPLY_DS;
    af2.modifier  = 20;
    af2.bitvector = 0;
    affect_to_char( victim, &af2 );
	
	af.type      = sn;
    af.duration  = level;

    af.location  = APPLY_AS;
    af.modifier  = -20;
	af.bitvector = 0;
    affect_to_char( victim, &af );
	}
	
	if( ch == victim )
	{
	act( "$n is surrounded by a churning barrier of air.", ch, NULL, NULL, TO_ROOM );
	act( "You are surrounded by a churning barrier of air.", ch, NULL, NULL, TO_CHAR );	
	}
	else
	{
	act( "$N is surrounded by a churning barrier of air.", ch, NULL, victim, TO_NOTVICT );
	act( "$N is surrounded by a churning barrier of air.", ch, NULL, victim, TO_CHAR );
	act( "You are surrounded by a churning barrier of air.", ch, NULL, victim, TO_VICT );	
	}
	
    return;
}
*/

void spell_strength( int sn, int level, CHAR_DATA *ch, void *vo )
{
	CHAR_DATA *victim = (CHAR_DATA *) vo;
    AFFECT_DATA af;

    if ( is_affected( victim, sn ) )
	{
	boost_duration( victim, sn, level );
	}
	else
	{
    af.type      = sn;
    af.duration  = level;
    af.location  = APPLY_AS;
    af.modifier  = 15;
    af.bitvector = 0;
    affect_to_char( victim, &af );
	}
	
	if( ch == victim )
	{
	act( "$n looks more physically imposing.", ch, NULL, NULL, TO_ROOM );
	act( "You feel stronger.", ch, NULL, NULL, TO_CHAR );	
	}
	else
	{
	act( "$N looks more physically imposing.", ch, NULL, victim, TO_NOTVICT );
	act( "$N looks more physically imposing.", ch, NULL, victim, TO_CHAR );
	act( "You feel stronger.", ch, NULL, victim, TO_VICT );	
	}
	
    return;
}

//thurfel's
void spell_protective_ward( int sn, int level, CHAR_DATA *ch, void *vo )
{
	CHAR_DATA *victim = (CHAR_DATA *) vo;
    AFFECT_DATA af;

    if ( is_affected( victim, sn ) )
	{
	boost_duration( victim, sn, level );
	}
	else
	{
    af.type      = sn;
    af.duration  = level;
    af.location  = APPLY_DS;
    af.modifier  = 15;
    af.bitvector = 0;
    affect_to_char( victim, &af );
	}
	
	if( ch == victim )
	{
	act( "$n is showered in a cascade of bright sparks, which quickly fades.", ch, NULL, NULL, TO_ROOM );
	act( "You are showered in a cascade of bright sparks, which quickly fades.", ch, NULL, NULL, TO_CHAR );	
	}
	else
	{
	act( "$N is showered in a cascade of bright sparks, which quickly fades.", ch, NULL, victim, TO_NOTVICT );
	act( "$N is showered in a cascade of bright sparks, which quickly fades.", ch, NULL, victim, TO_CHAR );
	act( "You are showered in a cascade of bright sparks, which quickly fades.", ch, NULL, victim, TO_VICT );	
	}
	
    return;
}

//healers.

void spell_heal_i( int sn, int level, CHAR_DATA *ch, void *vo )
{
	CHAR_DATA *victim = (CHAR_DATA *) vo;
	int w;
	int max=0;
	int b = -1;
	for( w = 0; w < MAX_BODY; w++ )
	{
	if( victim->body[w] == 1 && victim->body[w] > max )
	b = w;
	}
	
	if( b != -1 )
	{
	printf_to_char( victim, "The minor wounds on your %s heal.\r\n",
	body_name[b] );
	
	if( victim != ch )
	printf_to_char( ch, "You heal the minor wounds on %s's %s.\r\n", victim->name, body_name[b] );
	
	victim->body[b]--;
	
	if( victim->bleed[b] > 0 )
	{
	victim->bleed[b] = 0;
	victim->bandage[b] = 0;
	printf_to_char( victim, "Your %s stops bleeding.\r\n", body_name[b] );
	}
	
	if( victim->scars[b] == 0 )
	victim->scars[b] = 1;
	
	printf_to_char( victim, "Light scarring forms on your %s.\r\n", body_name[b] );
	
	act( "$n heals $N's minor wounds, leaving light scarring.", ch, NULL, victim, TO_NOTVICT );
	
	if( victim != ch )
	printf_to_char( ch, "Light scarring forms on %s's %s.\r\n", victim->name, body_name[b] );
	}
	else
		send_to_char( "Your skin flushes and you feel warm, but not much else.\r\n", victim );
	
    return;
}

void spell_heal_ii( int sn, int level, CHAR_DATA *ch, void *vo )
{
	CHAR_DATA *victim = (CHAR_DATA *) vo;
	int w;
	int max=0;
	int b = -1;
	for( w = 0; w < MAX_BODY; w++ )
	{
	if( victim->body[w] == 2 && victim->body[w] > max )
	b = w;
	}
	
	if( b != -1 )
	{
	printf_to_char( victim, "Your %s heals.\r\n",
	body_name[b] );
	

	if( victim != ch )
	printf_to_char( ch, "You heal %s's %s.\r\n", victim->name, body_name[b] );
	
	victim->body[b]--;
	
	if( victim->bleed[b] > 0 )
	{
	victim->bleed[b] = 0;
	victim->bandage[b] = 0;
	printf_to_char( victim, "Your %s stops bleeding.\r\n", body_name[b] );
	}
	
	if( victim->scars[b] < 2 )
	victim->scars[b] = 2;

	act( "$n heals $N's serious wounds, leaving some scarring.", ch, NULL, victim, TO_NOTVICT );
	
	printf_to_char( victim, "Scarring forms on your %s.\r\n",
	body_name[b] );
	
	if( victim != ch )
	printf_to_char( ch, "Scarring forms on %s's %s.\r\n", victim->name, body_name[b] );
	}
	else
		send_to_char( "Your skin flushes and you feel warm, but not much else.\r\n", victim );
	
    return;
}

void spell_heal_iii( int sn, int level, CHAR_DATA *ch, void *vo )
{
	CHAR_DATA *victim = (CHAR_DATA *) vo;
	int w;
	int max=0;
	int b = -1;
	for( w = 0; w < MAX_BODY; w++ )
	{
	if( victim->body[w] == 3 && victim->body[w] > max )
	b = w;
	}
	
	if( b != -1 )
	{
	printf_to_char( victim, "The critical wounds on your %s heal.\r\n",
	body_name[b] );
	
	if( victim != ch )
	printf_to_char( ch, "You heal the critical wounds on %s's %s.\r\n", victim->name, body_name[b] );
	
	victim->body[b]--;
	
	if( victim->bleed[b] > 0 )
	{
	victim->bleed[b] = 0;
	victim->bandage[b] = 0;
	printf_to_char( victim, "Your %s stops bleeding.\r\n", body_name[b] );
	}
	
	if( victim->scars[b] < 3 )
	victim->scars[b] = 3;

	act( "$n heals $N's critical wounds, leaving heavy scarring.", ch, NULL, victim, TO_NOTVICT );
	
	printf_to_char( victim, "Heavy scarring forms on your %s.\r\n",
	body_name[b] );
	
	if( victim != ch )
	printf_to_char( ch, "Heavy scarring forms on %s's %s.\r\n", victim->name, body_name[b] );
	}
	else
		send_to_char( "Your skin flushes and you feel warm, but not much else.\r\n", victim );
	
    return;
}

void spell_heal_scars_i( int sn, int level, CHAR_DATA *ch, void *vo )
{
	CHAR_DATA *victim = (CHAR_DATA *) vo;
	int w;
	int max=0;
	int b = -1;
	for( w = 0; w < MAX_BODY; w++ )
	{
	if( victim->scars[w] == 1 && victim->scars[w] > max )
	b = w;
	}
	
	if( b != -1 )
	{
	printf_to_char( victim, "The light scars on your %s fade away.\r\n",
	body_name[b] );
	
	act( "$n heals $N's light scars.", ch, NULL, victim, TO_NOTVICT );
	
	if( victim != ch )
	printf_to_char( ch, "You heal %s's lightly scarred %s.\r\n", victim->name, body_name[b] );
	
	victim->scars[b]--;
	}
	else
		send_to_char( "Your skin flushes and you feel warm, but not much else.\r\n", victim );
	
    return;
}

void spell_heal_scars_ii( int sn, int level, CHAR_DATA *ch, void *vo )
{
	CHAR_DATA *victim = (CHAR_DATA *) vo;
	int w;
	int max=0;
	int b = -1;
	for( w = 0; w < MAX_BODY; w++ )
	{
	if( victim->scars[w] == 2 && victim->scars[w] > max )
	b = w;
	}
	
	if( b != -1 )
	{
	printf_to_char( victim, "The scars on your %s fade away.\r\n",
	body_name[b] );
	
	act( "$n heals $N's scars.", ch, NULL, victim, TO_NOTVICT );
	
	if( victim != ch )
	printf_to_char( ch, "You heal %s's scarred %s.\r\n", victim->name, body_name[b] );
	
	victim->scars[b]--;
	}
	else
		send_to_char( "Your skin flushes and you feel warm, but not much else.\r\n", victim );
	
    return;
}

void spell_heal_scars_iii( int sn, int level, CHAR_DATA *ch, void *vo )
{
	CHAR_DATA *victim = (CHAR_DATA *) vo;
	int w;
	int max=0;
	int b = -1;
	for( w = 0; w < MAX_BODY; w++ )
	{
	if( victim->scars[w] == 3 && victim->scars[w] > max )
	b = w;
	}
	
	if( b != -1 )
	{
	printf_to_char( victim, "The heavy scars on your %s fade away.\r\n",
	body_name[b] );
	
	act( "$n heals $N's heavy scars.", ch, NULL, victim, TO_NOTVICT );
	
	if( victim != ch )
	printf_to_char( ch, "You heal %s's heavily scarred %s.\r\n", victim->name, body_name[b] );
	
	victim->scars[b]--;
	}
	else
		send_to_char( "Your skin flushes and you feel warm, but not much else.\r\n", victim );
	
    return;
}

void spell_restore_health_i( int sn, int level, CHAR_DATA *ch, void *vo )
{
	CHAR_DATA *victim = (CHAR_DATA *) vo;

	if( victim->max_hit == victim->hit )
	{
	if( victim != ch )
	printf_to_char(ch, "Your magic fizzles as you realize %s already has maximum health.\r\n", ch->name );

	send_to_char( "Your skin flushes and you feel warm, but not much else.\r\n", victim );
	return;
	}
	
	printf_to_char( victim, "Your skin flushes as your health is restored.\r\n" );
	
	if( victim != ch )
	printf_to_char( ch, "You heal %s's missing health.\r\n", victim->name );

	act( "$n restores $N's health.", ch, NULL, victim, TO_NOTVICT );
	
	victim->hit += number_range(10, 20); //fixme lore
	
	if( victim->hit > victim->max_hit );
	victim->hit = victim->max_hit;
	
    return;
}


void spell_restore_health_ii( int sn, int level, CHAR_DATA *ch, void *vo )
{
	CHAR_DATA *victim = (CHAR_DATA *) vo;

	if( victim->max_hit == victim->hit )
	{
	if( victim != ch )
	printf_to_char(ch, "Your magic fizzles as you realize %s already has maximum health.\r\n", ch->name );

	send_to_char( "Your skin flushes and you feel warm, but not much else.\r\n", victim );
	return;
	}
	
	printf_to_char( victim, "Your skin flushes as your health is restored.\r\n" );
	
	if( victim != ch )
	printf_to_char( ch, "You heal %s's missing health.\r\n", victim->name );
	
	victim->hit += number_range(40, 50); //fixme lore
	
	if( victim->hit > victim->max_hit );
	victim->hit = victim->max_hit;
	
    return;
}

void spell_resurrect( int sn, int level, CHAR_DATA *ch, void *vo )
{
	CHAR_DATA *victim = (CHAR_DATA *) vo;
	int w;
	int max=0;
	int b = -1;
	
        for( w = 0; w < MAX_BODY; w++ )
	{
	if( victim->body[w] < 4 && victim->body[w] > max )
	b = w;
	}
	
	//fixme? resurrect currently heals all injuries, IF you have a deed.
	if( b != -1 && !IS_NPC(victim) && victim->pcdata->deeds > 0  )
	{
	printf_to_char( victim, "Your injured %s is bathed in a white light as it is wholly restored.\r\n",
	body_name[b] );
	
	victim->body[b] = 0;
	}

	if( IS_SET( victim->affected_by, AFF_DEAD ) )
        {
        printf_to_char( victim, "You are resurrected!\r\n" );
        printf_to_char( ch, "You have resurrected %s.\r\n", victim->name );
	REMOVE_BIT( victim->affected_by, AFF_DEAD );

        victim->hit  = 1;
        victim->mana = 1;
        victim->move = 1;
        victim->position = POS_STANDING;
	victim->wait = 0;
	victim->stun = 0;
        }
		
		if( !IS_NPC(victim) )
			victim->pcdata->deeds--;
		
		if( !IS_NPC(victim) && victim->pcdata->deeds < 0 )
			victim->pcdata->deeds = 0;
	
    return;
}


void spell_spirit_guide( int sn, int level, CHAR_DATA *ch, void *vo )
{
	CHAR_DATA *victim = (CHAR_DATA *) vo;
    ROOM_INDEX_DATA *temple;

    if ( IS_NPC( victim ) )
	return;

    if ( (temple = get_room_index( ROOM_VNUM_TEMPLE ) ) == NULL )
    {
	send_to_char("Your spell fizzles and fails.\r\n", ch );
	return;
    }

    act("$n disappears.", ch, NULL, NULL, TO_ROOM );
    char_from_room( victim );
    char_to_room( victim, temple );
    do_look( victim, "auto" );
    act("$n appears in the center of the room.", ch, NULL, NULL, TO_ROOM );

    return;
}

//fixme is broken??

void spell_transference( int sn, int level, CHAR_DATA *ch, void *vo )
{
    ROOM_INDEX_DATA *to_room;
    ROOM_INDEX_DATA *in_room;
    int vnum;
    CHAR_DATA *victim;

    if ( (victim = get_char_world( ch, target_name ) ) == NULL )
    {
	send_to_char("Cast at whom?\n\r", ch );
	return;
    }
	
	to_room = victim->in_room;

    if ( to_room == NULL ) 
    {
	send_to_char("Your spell fizzles and fails.\n\r", ch );
	return;
    }
	
	vnum = to_room->vnum;
	
	if( vnum < 100 )
	{
	send_to_char("Your spell fails to gain access.\n\r", ch );
	return;
    }
	
    in_room = ch->in_room;
	
    if ( in_room == to_room )
    {
	send_to_char("You stop your casting when you realize you are already there.\n\r", ch );
	return;
    }
	
	if ( IS_NPC(victim) )
    {
	send_to_char("Cast at who?\n\r", ch );
	return;
    }

    act("$n disappears in a silver mist.", ch, NULL, NULL, TO_ROOM );
    char_from_room( ch );
    char_to_room( ch, to_room );
    act("$n arrives in a silver mist.", ch, NULL, NULL, TO_ROOM );
    send_to_char("You disappear in a silver mist.\n", ch );

    do_look( ch, "auto" );

    return;
}

//spells for sorcerers.

void spell_bleed( int sn, int level, CHAR_DATA *ch, void *vo )
{
	CHAR_DATA *victim = (CHAR_DATA *) vo;
	
	    short dam = warding_check( ch, victim );
		
	if( IS_SET(victim->affected_by, AFF_UNDEAD) || IS_SET(victim->affected_by, AFF_NONCORP) )
	{
		printf_to_char( ch, "Your sorcery requires the blood of the living.\r\n" );
		return;
	}
	
	if( dam != 0 )
	{
		short b = dam / 20;
		
		act( "   Blood slowly oozes out of $N's skin.", ch, NULL, victim, TO_NOTVICT );
		act( "   Blood slowly oozes out of $N's skin.", ch, NULL, victim, TO_CHAR );
		act( "   Blood slowly oozes out of your skin.", ch, NULL, victim, TO_VICT );	
		damage( ch, victim, dam * 111 / 1000, sn );
				
		if( b > 0 && !IS_DEAD( victim) )
		{
			short d = number_range(3,15);
			victim->bleed[d] += b;
			
			if( victim->body[d] < 2 )
				victim->body[d] = 2;
			
			printf_to_char( victim, "   Your %s starts bleeding at %d per!\r\n", body_name[d], b );
			act(  "   $n starts bleeding!", victim, NULL, NULL, TO_ROOM );
		}
	}
	
    return;
}

void spell_disrupt( int sn, int level, CHAR_DATA *ch, void *vo )
{
	CHAR_DATA *victim = (CHAR_DATA *) vo;
	
	    short dam = warding_check( ch, victim );
	
	if( dam != 0 )
	{
		act( "   $N's flesh starts to warp and bubble!", ch, NULL, victim, TO_NOTVICT );
		act( "   $N's flesh starts to warp and bubble!", ch, NULL, victim, TO_CHAR );
		act( "   Your flesh starts to warp and bubble!", ch, NULL, victim, TO_VICT );
		damage( ch, victim, dam * 222 / 1000, sn );		
	}
	
    return;
}

void spell_maelstrom( int sn, int level, CHAR_DATA *ch, void *vo )
{
	CHAR_DATA *vch;
    CHAR_DATA *vch_next;
	int dam;
	
	act( "$n creates a towering energy storm that crackles with power!", ch, NULL, NULL, TO_ROOM );
	act( "You create a towering energy storm that crackles with power!", ch, NULL, NULL, TO_CHAR );	

	
	//fixme really should create an object that lashes out periodically
    for ( vch = char_list; vch != NULL; vch = vch_next )
    {
	vch_next	= vch->next;
	if ( vch->in_room == NULL )
	    continue;
	if ( vch->in_room == ch->in_room )
	{
		
		if ( is_safe( ch, vch ) )
			continue;
		
	    if ( vch != ch && ( IS_NPC(ch) ? !IS_NPC(vch) : IS_NPC(vch) ) )
		{
			act( "A tendril of energy reaches out towards $N!", ch, NULL, vch, TO_NOTVICT );
			act( "A tendril of energy reaches out towards $N!", ch, NULL, vch, TO_CHAR );
			act( "A tendril of energy reaches out towards you!", ch, NULL, vch, TO_VICT );	
		
			dam = warding_check( ch, vch );
	
			if( dam != 0 )
			damage( ch, vch, dam * 300 / 1000, sn );
		}
	    continue;
	}

    }

    return;
}

void spell_curse( int sn, int level, CHAR_DATA *ch, void *vo )
{
    CHAR_DATA *victim;
    OBJ_DATA *obj;
    AFFECT_DATA af;

    if ( !targ_mob )
    {
	obj = (OBJ_DATA *) vo;
	
	if ( IS_OBJ_STAT(obj, ITEM_BLESS ) )
	{
		act( "A wave of darkness passes over $p, which quickly brightens.", ch, obj, NULL, TO_CHAR );
		return;
	}
	
	
	act("$p becomes dimmer and darker.", ch, obj, NULL, TO_CHAR );
	SET_BIT( obj->extra_flags, ITEM_NODROP|ITEM_NOREMOVE );
	return;
    }

    victim = (CHAR_DATA *) vo;

    if ( is_safe( ch, victim ) )
	return;

    if ( is_affected(victim, gsn_curse) )
	return;

    af.type      = gsn_curse;
    af.duration  = level * 5;
    af.location  = APPLY_TD;
    af.modifier  = -20;
    af.bitvector = 0;
    affect_to_char( victim, &af );

    send_to_char( "You suddenly feel vulnerable.\n\r", victim );
    act("$n briefly reveals a dark aura!", victim, NULL, NULL, TO_ROOM );
    return;
	
	return;
}

//fixme
void spell_animate_corpse( int sn, int level, CHAR_DATA *ch, void *vo )
{
	CHAR_DATA *victim = (CHAR_DATA *) vo;
    AFFECT_DATA af;
	
	if( !IS_DEAD(victim) )
	{
		send_to_char("But they're alive?\n\r", ch );
		send_to_char("You feel mildly invigorated.\n\r", victim );
		return;
	}
	
	REMOVE_BIT( victim->affected_by, AFF_DEAD );
	victim->hit = victim->max_hit;
	//undead don't bleed, so..we can leave wounds

    af.type      = sn;
    af.duration  = -1; //fixme
    af.location  = APPLY_NONE;
    af.modifier  = 0;
    af.bitvector = AFF_UNDEAD;
    affect_to_char( victim, &af );

    send_to_char("You breathe life back into the corpse.\n\r", ch );
    act("$n breathes life back into the corpse of $N.", ch, NULL, victim, TO_ROOM );


	//fixme warding check
    if ( number_range( 0, level ) > (victim->level - level + 1) )
    {
	add_follower( victim, ch );
	SET_BIT( victim->affected_by, AFF_CHARM );
	send_to_char("Success!  They are your undead slave now.\n\r", ch );
	send_to_char("You are now undead and bound to follow your new master.\n\r", victim );
    }
    else if( IS_NPC(victim) )
    {
	send_to_char("Your necromancy partially fails as the vile creature turns on you!\n\r", ch );
	printf_to_char(victim, "You hate %s for warping your soul!\n\r", ch->name );
	victim->hunting = ch;
    }
	else
	{
	send_to_char("You fail to bend them to your will.\n\r", ch );	
	//send_to_char("You are now undead.\n\r", victim );	
	}
	

    return;
}

void spell_bind_spirit ( int sn, int level, CHAR_DATA *ch, void *vo )
{
	//CHAR_DATA *victim = (CHAR_DATA *) vo;
    return;
}


void spell_summon_demon( int sn, int level, CHAR_DATA *ch, void *vo )
{
	//CHAR_DATA *victim = (CHAR_DATA *) vo;
    return;
}

void spell_sorcerous_travel( int sn, int level, CHAR_DATA *ch, void *vo )
{
    ROOM_INDEX_DATA *pRoomIndex;

    for ( ; ; )
    {
	pRoomIndex = get_room_index( number_range( 0, 65535 ) );
	if ( pRoomIndex != NULL )
	if ( !IS_SET(pRoomIndex->room_flags, ROOM_PRIVATE)
	&&   !IS_SET(pRoomIndex->room_flags, ROOM_SOLITARY) )
	    break;
    }

    act( "$n shimmers out of existence.", ch, NULL, NULL, TO_ROOM );
    char_from_room( ch );
    char_to_room( ch, pRoomIndex );
    act( "$n shimmers into existence.", ch, NULL, NULL, TO_ROOM );
    do_look( ch, "auto" );
    return;
}

void spell_manna_bread( int sn, int level, CHAR_DATA *ch, void *vo )
{
    OBJ_DATA *bread;

    bread = create_object( get_obj_index( OBJ_VNUM_MANNA_BREAD ), 0 );
    bread->value[0] = 5 + level;
	
	
	//fixme to hand - also, manna bread should do something
	
    obj_to_room( bread, ch->in_room );
    act( "$p suddenly appears and falls to the ground.", ch, bread, NULL, TO_ROOM );
    act( "$p suddenly appears and falls to the ground.", ch, bread, NULL, TO_CHAR );
    return;
}

//fixme bits etc needs to be less diku/merc
void spell_identify( int sn, int level, CHAR_DATA *ch, void *vo )
{
    OBJ_DATA *obj = (OBJ_DATA *) vo;
    AFFECT_DATA *paf;

    printf_to_char( ch, 
	"%s is type %s.\r\nIt weighs %d. It is worth %d.\r\nExtra flags: %s.\r\n",

	capitalize(obj->name),
	item_type_name( obj ),
	obj->weight,
	obj->cost,
    extra_bit_name( obj->extra_flags )
	);

    switch ( obj->item_type )
    {
    case ITEM_SCROLL:
	//fixme scrolls/ptoions may have multiple?
    case ITEM_POTION:
	
	if( get_gsn(obj->value[0]) == -1 )
	{
		send_to_char( "The item contains unknown magic.\r\n", ch );
		break;
	}
	
	printf_to_char( ch, "It contains the spell of %s.\r\n", spell_table[get_gsn(obj->value[0])].name );
	break;

//fixme sure this is wrong.
    case ITEM_WAND:
    case ITEM_STAFF:
	printf_to_char( ch, "Has %d(%d) charges of level %d",
	    obj->value[1], obj->value[2], obj->value[0] );

	if ( obj->value[3] >= 0 && obj->value[3] < MAX_SPELL )
	{
	    send_to_char( " '", ch );
	    send_to_char( spell_table[obj->value[3]].name, ch );
	    send_to_char( "'", ch );
	}

	send_to_char( ".\r\n", ch );
	break;

    case ITEM_WEAPON:
	printf_to_char( ch, "Weapon type: %s.\r\nAttack bonus: %d.\r\n", weapon_table[obj->value[0]].name, obj->value[2]);
	
	break;

//fixme armor type display
    case ITEM_ARMOR:
	printf_to_char( ch, "Armor values: %d %d %d %d.\r\n", obj->value[0], obj->value[1],
		obj->value[2], obj->value[3]	);
	break;
    }

    for ( paf = obj->pIndexData->affected; paf != NULL; paf = paf->next )
    {
	if ( paf->location != APPLY_NONE && paf->modifier != 0 )
	{
	    printf_to_char( ch, "Affects %s by %d.\r\n",
		affect_loc_name( paf->location ), paf->modifier );
	}
    }

    for ( paf = obj->affected; paf != NULL; paf = paf->next )
    {
	if ( paf->location != APPLY_NONE && paf->modifier != 0 )
	{
	    printf_to_char( ch,"Affects %s by %d.\r\n",
		affect_loc_name( paf->location ), paf->modifier );
	}
    }

    return;
}


void spell_bless_weapon( int sn, int level, CHAR_DATA *ch, void *vo )
{
    OBJ_DATA *obj = (OBJ_DATA *) vo;
    AFFECT_DATA *paf;
	
	// lots of stuff just can't be blessed.

    if ( obj->item_type != ITEM_WEAPON
	||	 IS_OBJ_STAT(obj, ITEM_MAGIC )
    ||   IS_OBJ_STAT(obj, ITEM_BLESS )
	||   IS_OBJ_STAT(obj, ITEM_NODROP )
	||   IS_OBJ_STAT(obj, ITEM_NOREMOVE )
    ||   obj->affected != NULL )
	{
		act( "A shower of sparks cascades over $p, which remains the same.", ch, obj, NULL, TO_ROOM );
		return;
	}

    if ( affect_free == NULL )
    {
	paf		= alloc_perm( sizeof(*paf) );
    }
    else
    {
	paf		= affect_free;
	affect_free	= affect_free->next;
    }

    paf->type		= -1;
    paf->duration	= -1;
    //paf->level		= level * 3;
    paf->location	= APPLY_NONE;
    paf->modifier	= level * 3; //swings

    paf->bitvector	= AFF_BLESS;
    paf->next		= obj->affected;
    obj->affected	= paf;

/* TO CHECK IF BLESSED
if (obj->affected && IS_SET( obj->affected->bitvector, AFF_BLESS ) && obj->affected->modifier > 0 )
{

}
*/

    act( "A blinding white light envelops $p for a moment and then appears to become one with it.", ch, obj, NULL, TO_CHAR );
    act( "A blinding white light envelops $p for a moment and then appears to become one with it.", ch, obj, NULL, TO_ROOM );

    SET_BIT( obj->extra_flags, ITEM_BLESS );

    //send_to_char( "Ok.\n\r", ch );
    return;
}


void spell_invis( int sn, int level, CHAR_DATA *ch, void *vo )
{
    CHAR_DATA *victim = (CHAR_DATA *) vo;
    AFFECT_DATA af;

    if ( IS_AFFECTED(victim, AFF_INVISIBLE) )
	return;

    act( "$n fades out of existence.", victim, NULL, NULL, TO_ROOM );
    af.type      = sn;
    af.duration  = level;
    af.location  = APPLY_NONE;
    af.modifier  = 0;
    af.bitvector = AFF_INVISIBLE;
    affect_to_char( victim, &af );
    send_to_char( "You fade out of existence.\n\r", victim );
    if ( ch != victim )
	send_to_char( "Ok.\n\r", ch );
    return;

/*
    obj = (OBJ_DATA *) vo;

    if ( IS_SET( obj->extra_flags, ITEM_INVIS ) )
	return;

    SET_BIT( obj->extra_flags, ITEM_INVIS );
    act("$p turns invisible.", ch, obj, NULL, TO_CHAR, TRUE );

    if ( obj->carried_by == NULL )
	act("$p turns invisible.", ch, obj, NULL, TO_ROOM, TRUE );
*/

    //return;
}


void spell_enchant( int sn, int level, CHAR_DATA *ch, void *vo )
{
send_to_char( "Your magic fails as you realize the spell is incomplete.\r\n", ch );
return;
}

void spell_summon_familiar( int sn, int level, CHAR_DATA *ch, void *vo )
{
send_to_char( "Your magic fails as you realize the spell is incomplete.\r\n", ch );
return;
}



void spell_mark( int sn, int level, CHAR_DATA *ch, void *vo )
{
if( IS_NPC(ch) )
{
send_to_char( "You cannot mark anyone.\r\n", ch );
return;
}
}

void spell_summon( int sn, int level, CHAR_DATA *ch, void *vo )
{
if( IS_NPC(ch) )
{
send_to_char( "You cannot summon anyone.\r\n", ch );
return;
}
}





void spell_null( int sn, int level, CHAR_DATA *ch, void *vo )
{
    send_to_char( "That's not a spell!\r\n", ch );
    return;
}


//fixme magic items, enchanted items, blessed items (for x swings)
void spell_detect_magic( int sn, int level, CHAR_DATA *ch, void *vo )
{
	AFFECT_DATA *paf;
	CHAR_DATA *victim = (CHAR_DATA *) vo;
	
	if( victim != ch )
	{
	printf_to_char( ch, "You detect the following spells affecting %s:\r\n", victim->name );
	if ( victim->affected != NULL )
    {
	for ( paf = victim->affected; paf != NULL; paf = paf->next )
	{
	    printf_to_char( ch, "%s\r\n", spell_table[paf->type].name );
	}
	}
	else
		printf_to_char( ch,"nothing\r\n" );
	}
	
	else
	{
	printf_to_char( victim, "You detect the following spells affecting you:\r\n" );
	
	if ( ch->affected != NULL )
    {
	for ( paf = ch->affected; paf != NULL; paf = paf->next )
	{
	    printf_to_char( ch, "%s\r\n", spell_table[paf->type].name );
	}
	}
	else
		printf_to_char( ch,"nothing\r\n" );
	}
	
    return;
}


CC      = gcc
PROF    =
C_FLAGS = -g3 -Wall $(PROF)
L_FLAGS = $(PROF)


O_FILES = act_comm.o act_info.o act_move.o act_obj.o act_wiz.o comm.o const.o \
          db.o fight.o handler.o interp.o magic.o save.o special.o update.o fileio.o mem_manage.o gamesys.o olc_room.o track.o stealth.o bank.o


merc: $(O_FILES)
	rm -f merc
	$(CC) -o merc $(O_FILES) $(L_FLAGS)

clean:
	rm *.o merc

.c.o: merc.h
	$(CC) -c $(C_FLAGS) $<


#if defined(macintosh)
#include <types.h>
#else
#include <sys/types.h>
#endif
#include <ctype.h>
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <time.h>
#include "merc.h"

#if !defined(macintosh)
extern	int	_filbuf		args( (FILE *) );
#endif


extern MOB_INDEX_DATA *        mob_index_hash          [MAX_KEY_HASH];
extern OBJ_INDEX_DATA *        obj_index_hash          [MAX_KEY_HASH];
extern int                     top_affect;
extern int                     top_area;
extern int                     top_ed;
extern int                     top_exit;
extern int                     top_help;
extern int                     top_mob_index;
extern int                     top_obj_index;
extern int                     top_reset;
extern int                     top_room;
extern int                     top_shop;

/*
 * Locals.
 */
 
char *			string_space;
char *			top_string;
char			str_empty	[1];

/*
 * Memory management.
 * Increase MAX_STRING if you have too.  (moved to merc.h)
 * Tune the others only if you understand what you're doing.
 */
#define			MAX_PERM_BLOCK	131072
#define			MAX_MEM_LIST	11

void *			rgFreeList	[MAX_MEM_LIST];
const int		rgSizeList	[MAX_MEM_LIST]	=
{
    16, 32, 64, 128, 256, 1024, 2048, 4096, 8192, 16384, 32768-64
};

int			nAllocString;
int			sAllocString;
int			nAllocPerm;
int			sAllocPerm;

struct mem_hdr
{
  unsigned int magic;
  int references;
};

void init_string_space( void )
{
  if ( ( string_space = (char *) calloc( 1, MAX_STRING ) ) == NULL )
  {
    bug( "init_string_space: can't alloc %d string space.", MAX_STRING );
    exit( 1 );
  }
  top_string	= string_space;

  return;
}


void *alloc_mem( int sMem )
{
    void *pMem;
    int iList;

    for ( iList = 0; iList < MAX_MEM_LIST; iList++ )
    {
	if ( sMem <= rgSizeList[iList] )
	    break;
    }

    if ( iList == MAX_MEM_LIST )
    {
	bug( "Alloc_mem: size %d too large.", sMem );
	exit( 1 );
    }

    if ( rgFreeList[iList] == NULL )
    {
	pMem		  = alloc_perm( rgSizeList[iList] );
    }
    else
    {
	pMem              = rgFreeList[iList];
	rgFreeList[iList] = * ((void **) rgFreeList[iList]);
    }

    return pMem;
}






void free_mem( void *pMem, int sMem )
{
    int iList;

    for ( iList = 0; iList < MAX_MEM_LIST; iList++ )
    {
	if ( sMem <= rgSizeList[iList] )
	    break;
    }

    if ( iList == MAX_MEM_LIST )
    {
	bug( "Free_mem: size %d too large.", sMem );
	exit( 1 );
    }

    * ((void **) pMem) = rgFreeList[iList];
    rgFreeList[iList]  = pMem;

    return;
}



/*
 * Allocate some permanent memory.
 * Permanent memory is never freed,
 *   pointers into it may be copied safely.
 */
void *alloc_perm( int sMem )
{
  static char *pMemPerm = NULL;
  static int iMemPerm = 0;
  void *pMem;

  while ( sMem % sizeof(long) != 0 )
    sMem++;

  if ( sMem > MAX_PERM_BLOCK )
  {
    bug( "Alloc_perm: %d too large.", sMem );
      exit( 1 );
  }

  if ( pMemPerm == NULL || iMemPerm + sMem > MAX_PERM_BLOCK )
  {
    iMemPerm = 0;
    if ( ( pMemPerm = (char *) calloc( 1, MAX_PERM_BLOCK ) ) == NULL )
    {
      perror( "Alloc_perm" );
      exit( 1 );
    }
  }

  pMem        = pMemPerm + iMemPerm;
  iMemPerm   += sMem;
  nAllocPerm += 1;
  sAllocPerm += sMem;
  return pMem;
}




/*
 * Duplicate a string into dynamic memory.
 * Fread_strings are read-only and shared.
 */
char *str_dup( const char *str )
{
    char *str_new;

    if ( str[0] == '\0' )
	return &str_empty[0];

    if ( str >= string_space && str < top_string )
	return (char *) str;

    str_new = alloc_mem( strlen(str) + 1 );
    strcpy( str_new, str );
    return str_new;
}



/*
 * add or strip tabs.
 * fStrip is whether tabs are stripped or placed.
 */
char *str_dup_tab( const char *str, bool fStrip )
{
  char *str_new;
  char buf[MAX_STRING_LENGTH];
  char *pstr, *pbuf;
  char *endstr = (char *) ( str + strlen( str ) );

  if ( str[0] == '\0' )
    return &str_empty[0];

  pstr = (char *) str;
  pbuf = buf;
  while ( pstr < endstr ) 
  {
    if ( pstr[0] == '\t' && fStrip )
    {
      strncpy( pbuf, "     ", 5 );
      pbuf = &pbuf[5];
      pstr++;
      continue;
    }

    if ( pstr[0] == ' ' && !fStrip && &pstr[4] < endstr
    && pstr[1] == ' ' &&  pstr[2] == ' ' && pstr[3] == ' ' && pstr[4] == ' ' )
    {
      pbuf[0] = '\t';
      pbuf++;
      pstr = &pstr[5];
      continue;
    }

    pbuf[0] = pstr[0];
    pbuf++;
    pstr++;
  }

  pbuf[0] = '\0';

  str_new = (char *) alloc_mem( strlen(buf) + 1 );
  strcpy( str_new, buf );
  return str_new;
}



/*
 * Free a string.
 * Null is legal here to simplify callers.
 * Read-only shared strings are not touched.
 */
#if 0
void free_string( char *pstr )
{
  if ( pstr == NULL
  ||   pstr == &str_empty[0]
  || ( pstr >= string_space && pstr < top_string ) )
    return;

  /* Free mem now handles reference counts */
  free_mem( pstr, strlen(pstr) + 1 );
  return;
}
#endif

void free_string( char *pstr )
{
    if ( pstr == NULL
    ||   pstr == &str_empty[0]
    || ( pstr >= string_space && pstr < top_string ) )
	return;

    free_mem( pstr, strlen(pstr) + 1 );
    return;
}



void do_memory( CHAR_DATA *ch, char *argument )
{
    char buf[MAX_STRING_LENGTH];

    sprintf( buf, "Affects %5d\n\r", top_affect    ); send_to_char( buf, ch );
    sprintf( buf, "Areas   %5d\n\r", top_area      ); send_to_char( buf, ch );
    sprintf( buf, "ExDes   %5d\n\r", top_ed        ); send_to_char( buf, ch );
    sprintf( buf, "Exits   %5d\n\r", top_exit      ); send_to_char( buf, ch );
    sprintf( buf, "Helps   %5d\n\r", top_help      ); send_to_char( buf, ch );
    sprintf( buf, "Mobs    %5d\n\r", top_mob_index ); send_to_char( buf, ch );
    sprintf( buf, "Objs    %5d\n\r", top_obj_index ); send_to_char( buf, ch );
    sprintf( buf, "Resets  %5d\n\r", top_reset     ); send_to_char( buf, ch );
    sprintf( buf, "Rooms   %5d\n\r", top_room      ); send_to_char( buf, ch );
    sprintf( buf, "Shops   %5d\n\r", top_shop      ); send_to_char( buf, ch );

    sprintf( buf, "Strings %5d strings of %7d bytes (max %d).\n\r",
	nAllocString, sAllocString, MAX_STRING );
    send_to_char( buf, ch );

    sprintf( buf, "Perms   %5d blocks  of %7d bytes.\n\r",
	nAllocPerm, sAllocPerm );
    send_to_char( buf, ch );

    return;
}
// Notes: none


#define crypt(s1, s2) (s1)

 
 
#define args( list )			list
#define DECLARE_DO_FUN( fun )		DO_FUN    fun
#define DECLARE_SPEC_FUN( fun )		SPEC_FUN  fun
#define DECLARE_SPELL_FUN( fun )	SPELL_FUN fun
#define DECLARE_SPEC_FUN( fun )		SPEC_FUN  fun
#define DECLARE_ROOM_SPECIAL( fun )	ROOM_SPECIAL fun
#define DECLARE_OBJ_FUN( fun )		OBJ_FUN  fun

/*
 * Short scalar type
 * Diavolo reports AIX compiler has bugs with short types.
 */
#if	!defined(FALSE)
#define FALSE	 0
#endif

#if	!defined(TRUE)
#define TRUE	 1
#endif

typedef unsigned char			bool;

/*
 * Structure types.
 */
typedef struct	affect_data		AFFECT_DATA;
typedef struct	area_data		AREA_DATA;
typedef struct	ban_data		BAN_DATA;
typedef struct	char_data		CHAR_DATA;
typedef struct	descriptor_data		DESCRIPTOR_DATA;
typedef struct	equip_data		EQUIP_DATA;
typedef struct	exit_data		EXIT_DATA;
typedef struct	extra_descr_data	EXTRA_DESCR_DATA;
typedef struct	help_data		HELP_DATA;
typedef struct	kill_data		KILL_DATA;
typedef struct	mob_index_data		MOB_INDEX_DATA;
typedef struct	note_data		NOTE_DATA;
typedef struct	obj_data		OBJ_DATA;
typedef struct	obj_index_data		OBJ_INDEX_DATA;
typedef struct	pc_data			PC_DATA;
typedef struct	reset_data		RESET_DATA;
typedef struct	room_index_data		ROOM_INDEX_DATA;
typedef struct	shop_data		SHOP_DATA;
typedef struct	time_info_data		TIME_INFO_DATA;
typedef struct	weather_data		WEATHER_DATA;
typedef	struct	skill_delay_data	SKILL_DELAY_DATA;

typedef struct	predelay_data		PREDELAY_DATA;
typedef struct	verb_trap_data	    VERB_TRAP_DATA;

typedef struct quest_data QUEST_DATA;

/*
 * Function types.
 */
typedef	void DO_FUN	args( ( CHAR_DATA *ch, char *argument ) );
typedef bool OBJ_FUN	args( ( CHAR_DATA *ch, char *cmd, DO_FUN * fnptr, char *argument, OBJ_DATA *obj ) );
//typedef bool SPEC_FUN	args( ( CHAR_DATA *ch ) );
typedef bool SPEC_FUN	args( ( CHAR_DATA *ch, char *cmd, DO_FUN * fnptr, char *argument, CHAR_DATA *mob ) );
typedef bool ROOM_SPECIAL args( ( CHAR_DATA *ch, char *cmd, DO_FUN * fnptr, char *argument, ROOM_INDEX_DATA *room ) );
typedef void SPELL_FUN	args( ( int sn, int level, CHAR_DATA *ch, void *vo ) );




/*
 * String and memory management parameters.
 */
#define	MAX_KEY_HASH		 1024
#define MAX_STRING_LENGTH	 4096
#define MAX_INPUT_LENGTH	  160
#define MAX_STRING            1048576


/*
 * Game parameters.
 * Increase the max'es if you add more of something.
 * Adjust the pulse numbers to suit yourself.
 */

#define GAME_NAME "GrindStone"
#define GAME_VERSION "0.3.1"

#define MAX_REAL_WEAPON		14
#define MAX_WEAPON			24
#define MAX_BOLT_SPELL		10

#define MAX_SHIELD 10
	
#define RACE_HUMAN 0
#define RACE_HIGH_MAN 1
#define RACE_HALF_ELF 2
#define RACE_HIGH_ELF 3
#define RACE_WOOD_ELF 4
#define RACE_DARK_ELF 5
#define RACE_DWARF 6
#define RACE_HALFLING 7
#define RACE_GNOME 8

//CLASSLESS FOR PLAYER CHARACTERS NOW! - NMS 11/16/2020
#define CLASS_FIGHTER 0
#define CLASS_THIEF 1
#define CLASS_PRIEST 2
#define CLASS_MAGE 3
#define CLASS_HEALER 4
#define CLASS_SORCERER 5
#define CLASS_RANGER 6
#define CLASS_BARD 7
#define CLASS_PALADIN 8
#define CLASS_MONK 9
#define CLASS_SAVANT 10

#define MANEUVER_AVERAGE 	0
#define MANEUVER_GOOD    	3
#define MANEUVER_EXCELLENT 	6
 
#define MAX_SKILL		   		34  // from 37 due to descaling of magic.
#define MAX_SPELL		   		50
#define MAX_CLASS		    	6 // for NPCs only due to classless design now.
#define MAX_RACE		    	9

#define MAX_FACTION 5
#define MAX_QUEST 5

#define MAX_LEVEL		   		110
#define LEVEL_HERO		   	   (MAX_LEVEL - 10)
#define LEVEL_IMMORTAL		   (MAX_LEVEL - 9)
#define EXP_BOOST				3

#define PULSE_PER_SECOND	    4
#define PULSE_TICK		  (120 * PULSE_PER_SECOND)
#define PULSE_AREA		  (2 * PULSE_PER_SECOND)
#define PULSE_WEATHER		( 60 * PULSE_PER_SECOND )

#define	PULSE_RAPID_UPDATE		PULSE_PER_SECOND+1
#define RAPID_BEAT		1


/*
 * Site ban structure.
 */
struct	ban_data
{
    BAN_DATA *	next;
    char *	name;
};



/*
 * Time and weather stuff.
 */
#define SUN_DARK		    0
#define SUN_RISE		    1
#define SUN_LIGHT		    2
#define SUN_SET			    3

#define SKY_CLOUDLESS		    0
#define SKY_CLOUDY		    1
#define SKY_RAINING		    2
#define SKY_LIGHTNING		    3

struct	time_info_data
{
    int		hour;
    int		day;
    int		month;
    int		year;
};

struct	weather_data
{
	int		temp;
	int		precip_start_hour;
	int		precip_start_minute;
	int		precip_end_hour;
	int		precip_end_minute;
	int		todays_high;
	int		tonights_low;
    int		sky;
    int		sunlight;
};


/*
 * A skill delay.
 */
struct	skill_delay_data
{
    SKILL_DELAY_DATA *	next;
    short		skill;
    short		duration;
};




/* ansi/vt100 */
#define CLEAR                   "^[[0;37m"
#define C_WHITE                 CLEAR
#define C_RED                   "^[[0;31m"
#define C_GREEN                 "^[[0;32m"
#define C_YELLOW                "^[[0;33m"
#define C_BLUE                  "^[[0;34m"
#define C_MAGENTA               "^[[0;35m"
#define C_CYAN                  "^[[0;36m"
#define C_DARK                  "^[[1;30m"
#define C_B_RED                 "^[[1;31m"
#define C_B_GREEN               "^[[1;32m"
#define C_B_YELLOW              "^[[1;33m"
#define C_B_BLUE                "^[[1;34m"
#define C_B_MAGENTA             "^[[1;35m"
#define C_B_CYAN                "^[[1;36m"
#define C_B_WHITE               "^[[1;37m"
#define TEXT_BOLD_ON            "\033[0;1m"
#define TEXT_NORMAL     CLEAR
#define VT100_CURS_UP           "\033[%dA"
#define VT100_CURS_DOWN         "\033[%dB"
#define VT100_CURS_RIGHT        "\033[%dC"
#define VT100_CURS_LEFT         "\033[%dD"
#define VT100_CURS_POSITION     "\033[%d;%dH"
#define VT100_CURS_HOME         "\033[H"
#define VT100_DEOL_X            "\033[K"
#define VT100_DEOL_0            "\033[0K"
#define VT100_D_LINETOCURS      "\033[1K"
#define VT100_DLINE             "\033[2K"
#define VT100_E_CURSE_TO_BOT    "\033[J"
#define VT100_E_CURSE_TO_BOT0   "\033[0J"
#define VT100_E_TOP_TO_CURS     "\033[1J"
#define VT100_ERASE_SCREEN      "\033[2J"

#define SAY						"\e[1;32m"
#define WHISPER					"\e[1;36m"
#define CHAT					"\e[0;35m"
#define ESP						"\e[1;34m"

#define COMMUNICATION			"\e[1;32m"
#define MONSTERBOLD				"\e[1;33m"
#define ROOMTITLE               "\e[1;37m"
#define NORMAL					"\e[0;0m"

#define ANSI_BEEP	"\007"


/*

#define C_BLACK		"\033[0;30m"
#define C_RED		"\033[0;31m"
#define C_GREEN		"\033[0;32m"
#define C_YELLOW	"\033[0;33m"
#define C_BLUE		"\033[0;34m"
#define C_MAGENTA	"\033[0;35m"
#define C_CYAN		"\033[0;36m"
#define C_WHITE		"\033[0;37m"
#define C_B_BLACK	"\033[1;30m"
#define C_B_RED		"\033[1;31m"
#define C_B_GREEN	"\033[1;32m"
#define C_B_YELLOW	"\033[1;33m"
#define C_B_BLUE	"\033[1;34m"
#define C_B_MAGENTA	"\033[1;35m"
#define C_B_CYAN	"\033[1;36m"
#define C_B_WHITE	"\033[1;37m"
#define ANSI_OFF	"\033[0m"
#define ANSI_BOLD	"\033[0;1m"
#define ANSI_UNDERSCORE	"\033[0;4m"
#define ANSI_BLINK	"\033[0;5m"
#define ANSI_REVERSE    "\033[0;7m"
#define ANSI_CONCEALED  "\033[0;8m"
#define ANSI_BEEP	"\007"
*/


//fixme channels

#define CON_PLAYING			 0
#define CON_GET_NAME			 1
#define CON_GET_OLD_PASSWORD		 2
#define CON_CONFIRM_NEW_NAME		 3
#define CON_GET_NEW_PASSWORD		 4
#define CON_CONFIRM_NEW_PASSWORD	 5
#define CON_GET_NEW_SEX			 6
#define CON_GET_NEW_RACE		 7
#define CON_GET_NEW_CLASS		 8
#define CON_GET_NEW_HOMETOWN		 9
#define CON_READ_MOTD			 10

#define CON_MENU			100

#define CON_BUFFER			200

/*
 * Descriptor (channel) structure.
 */
struct	descriptor_data
{
    DESCRIPTOR_DATA *	next;
    DESCRIPTOR_DATA *	snoop_by;
    CHAR_DATA *		character;
    CHAR_DATA *		original;
    char *		host;
    short		descriptor;
    short		connected;
    bool		fcommand;
    char		inbuf		[4 * MAX_INPUT_LENGTH];
    char		incomm		[MAX_INPUT_LENGTH];
    char		inlast		[MAX_INPUT_LENGTH];
    int			repeat;
    char *		outbuf;
    int			outsize;
    int			outtop;
};


/*
 * Predelay data.  Used to pass arguments to chained functions.
 */
struct predelay_data
{
    DO_FUN *		fnptr;
    char		argument[MAX_INPUT_LENGTH];
    int			number;
    CHAR_DATA *		victim1;
    CHAR_DATA *		victim2;
    OBJ_DATA *		obj1;
    OBJ_DATA *		obj2;
	bool		noInterrupt;
};




/*
 * TO types for act.
 */
#define TO_ROOM		    0
#define TO_NOTVICT	    1
#define TO_VICT		    2
#define TO_CHAR		    3



/*
 * Help table types.
 */
struct	help_data
{
    HELP_DATA *	next;
    short	level;
    char *	keyword;
    char *	text;
};



/*
 * Shop types.
 */
#define MAX_TRADE	 5

struct	shop_data
{
    SHOP_DATA *	next;			/* Next shop in list		*/
    short	keeper;			/* Vnum of shop keeper mob	*/
    short	buy_type [MAX_TRADE];	/* Item types shop will buy	*/
    short	profit_buy;		/* Cost multiplier for buying	*/
    short	profit_sell;		/* Cost multiplier for selling	*/
    short	open_hour;		/* First opening hour		*/
    short	close_hour;		/* First closing hour		*/
};



/*
 * Per-class stuff.
 */
struct	class_type
{
    char 	*name;
	short	skillEmphasis;
	short	firstStat;
	short	secondStat;
	//spell circles
	short	firstSphere;
	short	secondSphere;
	short	thirdSphere;
};





// race data
struct	race_type
{
	char *		name;
	short		lifespan;
	char *		hair;
	char *		eyes;
	char *		skin;
	short		max_Health;
	short		weight_Factor; // also base weight
	short		encumbrance_Factor;
	short		maneuver_Bonus;
	short		HP_per_level;
	short		grow_Strength;
	short		grow_Endurance;
	short		grow_Dexterity;
	short		grow_Speed;
	short		grow_Willpower;
	short		grow_Potency;
	short		grow_Judgement;
	short		grow_Intelligence;
	short		grow_Wisdom;
	short		grow_Charm;
	int			body_parts;
};

struct	box_matl_type
{
	char *name;
	short flags;
};

struct	box_adj_type
{
	char *name;
	short flags;

};

struct	box_noun_type
{
	char *name;
	short flags;

};

struct	container_matl_type
{
	char *name;
	short flags;
};

struct	container_adj_type
{
	char *name;
	short flags;

};

struct	container_noun_type
{
	char *name;
	int flags;

};


struct	magic_item_type
{
	char *name;
	short item_type;
	short charges;
	short spell;
	short weight;
	short cost;
};

struct	metal_type
{
	char *name;
	short weight_modifier;
	short cost_modifier;
	short enchant;
	short st_modifier;
	short du_modifier;
};

struct	armor_type
{
	char *name;
	long worn;
	short st;
	short du;
	short v0;
	short v1;
	short v2;
	short v3;
	short weight;
	short cost;
	short RT;
	short AP;
	short CvA1;
	short CvA2;
	short Hindrance;
	short HindranceMax;
};

struct	gem_type
{
	char *name;
	short tier;
	short min;
	short max;
};

struct	creature_type
{
	char *name;
	short level;
	short ob;
	short db;
	short cs;
	short td;
	short hp;
	int act;
	int aff;
	short unused6;
	short class;
	char *skin;
	short unused7;
};

// shield data
struct	shield_type
{
	char *name;
	short st;
	short du;
	short size;
	short weight;
	short cost;
};

// weapon data
struct	weapon_type
{
	char *name;
	short st;
	short du;
	short weapon_type;
	short weapon_speed;
	short damage_factor_skin;
	short damage_factor_leather;
	short damage_factor_scale;
	short damage_factor_chain;
	short damage_factor_plate;
	short weight;
	short cost;
};

struct	bolt_type
{
	char *name;
	short damage_type;
	short damage_factor_skin;
	short damage_factor_leather;
	short damage_factor_scale;
	short damage_factor_chain;
	short damage_factor_plate;
};

struct	crit_type
{
	char *msg;
	short damage;
	short status;
	short wound;
	short bleed;
};


struct	faction_type
{
	char *name;
	short favor;
};


struct	quest_type
{
	char *name;
	short reward;
};

/*
 * Data structure for notes.
 */
struct	note_data
{
    NOTE_DATA *	next;
    char *	sender;
    char *	date;
    char *	to_list;
    char *	subject;
    char *	text;
};



/*
 * An affect.
 */
struct	affect_data
{
    AFFECT_DATA *	next;
    short		type;
    short		duration;
    short		location;
    short		modifier;
    int			bitvector;
};



/*
 * A kill structure (indexed by level).
 */
struct	kill_data
{
    short		number;
    short		killed;
};



/***************************************************************************
 *                                                                         *
 *                   VALUES OF INTEREST TO AREA BUILDERS                   *
 *                   (Start of section ... start here)                     *
 *                                                                         *
 ***************************************************************************/




/*
 * ACT bits for mobs.
 * Used in #MOBILES.
 */
#define ACT_IS_NPC		      1		/* Auto set for mobs	*/
#define ACT_SENTINEL		      2		/* Stays in one room	*/
#define ACT_SCAVENGER		      4		/* Picks up objects	*/
#define ACT_AGGRESSIVE		     32		/* Attacks PC's		*/
#define ACT_STAY_AREA		     64		/* Won't leave area	*/
#define ACT_TOWN		    128		//replaces ACT_WIMPY
#define ACT_PET			    256		/* Auto set for pets	*/
#define ACT_TRAIN		    512		/* Can train PC's	*/
#define ACT_PRACTICE		   1024		/* Can practice PC's	*/

#define ACT_TERMINATOR		ACT_TRAIN

#define ACT_BITE			2048
#define ACT_CHARGE			4096
#define ACT_CLAW			8192
#define ACT_POUND			16384

#define ACT_KILL_AGGRO		32768

#define ACT_ANIMAL		    65536

#define ACT_SPELLCASTER		32768*4

#define ACT_SEXLESS			ACT_SPELLCASTER*2

#define ACT_AUTOMATON		ACT_SEXLESS*2


/*
 * Bits for 'affected_by'.
 * Used in #MOBILES.
 */
#define AFF_BLIND		      1
#define AFF_INVISIBLE		      2
#define AFF_DETECT_EVIL		      4
#define AFF_DETECT_INVIS	      8
#define AFF_DETECT_MAGIC	     16
#define AFF_DETECT_HIDDEN	     32
#define AFF_DEAD				64
#define AFF_SANCTUARY		    128
#define AFF_HOLD			AFF_SANCTUARY //repurposing
#define AFF_FAERIE_FIRE		    256
#define AFF_SKINNED			AFF_FAERIE_FIRE	//repurposing
#define AFF_INFRARED		    512
#define AFF_MARKED			AFF_INFRARED	// repurposing
#define AFF_CURSE		   1024
#define AFF_UNDEAD		   2048		
#define AFF_POISON		   4096
#define AFF_PROTECT		   8192
#define AFF_NONCORP		  16384		
#define AFF_SNEAK		  32768
#define AFF_HIDE		  65536
#define AFF_SLEEP		 131072
#define AFF_CHARM		 262144
#define AFF_FLYING		 524288
#define AFF_PASS_DOOR		1048576
#define AFF_BLESS			1048576*2
#define AFF_TRAPPED			AFF_BLESS*2


//currently unused
#define SPEED_FAST			3
#define SPEED_NORMAL		5
#define SPEED_SLOW			7


/*
 * Sex.
 * Used in #MOBILES.
 */
#define SEX_NEUTRAL		      0
#define SEX_MALE		      1
#define SEX_FEMALE		      2



/*
 * Well known object virtual numbers.
 * Defined in #OBJECTS.
 */
#define OBJ_VNUM_MONEY_ONE	      2
#define OBJ_VNUM_MONEY_SOME	      3

#define OBJ_VNUM_SHIELD			  3354
#define OBJ_VNUM_GEM			  7
#define OBJ_VNUM_ARMOR			  3353
#define OBJ_VNUM_WEAPON			  3350
#define OBJ_VNUM_WAND                     3401
#define OBJ_VNUM_POTION_FLASK		3402

#define OBJ_VNUM_SKIN			100

#define OBJ_VNUM_CORPSE_NPC	     10
#define OBJ_VNUM_CORPSE_PC	     11

#define OBJ_VNUM_MANNA_BREAD     3009 //fixme

#define OBJ_VNUM_LOCKPICK 		45





/*
 * Item types.
 * Used in #OBJECTS.
 */
#define ITEM_LIGHT		      1
#define ITEM_SCROLL		      2
#define ITEM_WAND		      3
#define ITEM_STAFF		      4
#define ITEM_WEAPON		      5
#define	ITEM_SHIELD			  6
#define ITEM_GEM		  7
#define ITEM_TREASURE		      8
#define ITEM_ARMOR		      9
#define ITEM_POTION		     10
#define ITEM_RUB		 11
#define ITEM_FURNITURE		     12
#define ITEM_TRASH		     13
#define	ITEM_UNUSED4		 14
#define ITEM_CONTAINER		     15
#define ITEM_SKIN			 16
#define ITEM_DRINK_CON		     17
#define ITEM_KEY		     18
#define ITEM_FOOD		     19
#define ITEM_MONEY		     20
#define ITEM_BOAT		     22
#define ITEM_CORPSE_NPC		     23
#define ITEM_CORPSE_PC		     24
#define ITEM_FOUNTAIN		     25
#define ITEM_PILL		     26
#define	ITEM_PORTAL			 27
#define ITEM_BUILDING		 28
#define ITEM_DOOR			 29
#define ITEM_CLIMB			 30
#define	ITEM_SWIM			 31
#define ITEM_NEST			 32
#define ITEM_LOCKPICK        33
#define ITEM_MERCHANT_TABLE  34


/*
 * Extra flags.
 * Used in #OBJECTS.
 */
#define ITEM_VT_ROOM		  1
#define ITEM_VT_INVENTORY	  2
#define ITEM_DARK		      4
#define ITEM_LOCK		      8
#define ITEM_EVIL		     16
#define ITEM_INVIS		     32
#define ITEM_MAGIC		     64
#define ITEM_NODROP		    128
#define ITEM_BLESS		    256
#define ITEM_GENERATED		    512
#define ITEM_ANTI_EVIL		   1024
#define ITEM_ANTI_NEUTRAL	   2048
#define ITEM_NOREMOVE		   4096
#define ITEM_INVENTORY		   8192
#define ITEM_HIDDEN			16384
#define ITEM_TRAPPED		32768

/*
 * Wear flags.
 * Used in #OBJECTS.
 */
#define ITEM_EQUIP_TAKE			1
#define ITEM_EQUIP_RIGHTHAND 	2
#define ITEM_EQUIP_LEFTHAND		4
#define ITEM_EQUIP_HEAD			8
#define ITEM_EQUIP_EARS			16
#define ITEM_EQUIP_NECK			32
#define ITEM_EQUIP_AROUND_NECK 	64
#define ITEM_EQUIP_BACK			128
#define ITEM_EQUIP_ARMS			256
#define ITEM_EQUIP_HANDS		512
#define ITEM_EQUIP_WRIST		1024
#define ITEM_EQUIP_FRONT		2048
#define ITEM_EQUIP_CHEST		4096
#define ITEM_EQUIP_FINGER		8192
#define ITEM_EQUIP_BELT			16384
#define ITEM_EQUIP_ONBELT		32768
#define ITEM_EQUIP_LEGS			65536
#define ITEM_EQUIP_ANKLE		131072
#define ITEM_EQUIP_FEET			262144
#define ITEM_EQUIP_PIN			524288
#define ITEM_EQUIP_SHOULDER		1045876
#define ITEM_EQUIP_CLOAK		2097152

#define SHIELD_SMALL 0
#define SHIELD_MEDIUM 1
#define SHIELD_LARGE 2
#define SHIELD_TOWER 3

// weapons

#define WEAPON_TYPE_NONE		0
#define WEAPON_TYPE_EDGED		1
#define WEAPON_TYPE_BLUNT		2
#define WEAPON_TYPE_TWOHANDED	3
#define WEAPON_TYPE_POLEARM		4
#define WEAPON_TYPE_BRAWLING	5
#define WEAPON_TYPE_RANGED		6
#define WEAPON_TYPE_THROWN		7
#define WEAPON_TYPE_NATURAL 	8

#define WEAPON_SPEED_NONE		0
#define WEAPON_SPEED_VERY_FAST	3
#define WEAPON_SPEED_FAST		4
#define WEAPON_SPEED_NORMAL		5
#define WEAPON_SPEED_SLOW		6
#define WEAPON_SPEED_VERY_SLOW	7

#define WEAPON_FIRE_FLARES 66

// armor flags

#define ARMOR_GROUP_SKIN 	0
#define ARMOR_GROUP_LEATHER 1
#define ARMOR_GROUP_SCALE 	2
#define ARMOR_GROUP_CHAIN 	3
#define ARMOR_GROUP_PLATE 	4
#define MAX_ARMOR_GROUP		5

#define ARMOR_COVERS_EYES	2
#define ARMOR_COVERS_HEAD	4
#define ARMOR_COVERS_NECK	8
#define ARMOR_COVERS_TORSO	16
#define ARMOR_COVERS_ARMS	32
#define ARMOR_COVERS_HANDS	64
#define ARMOR_COVERS_LEGS	128
#define ARMOR_COVERS_FEET	256

// body flags

#define BODY_NONE	0
#define BODY_R_EYE	1
#define BODY_L_EYE	2
#define BODY_HEAD 	3
#define BODY_NECK	4
#define BODY_CHEST	5
#define BODY_BACK	6
#define BODY_R_ARM	7
#define BODY_R_HAND	8
#define BODY_L_ARM	9
#define BODY_L_HAND	10
#define BODY_ABDOMEN 11
#define BODY_R_LEG	12
#define BODY_R_FOOT 13
#define BODY_L_LEG	14
#define	BODY_L_FOOT	15
#define BODY_CNS	16
#define MAX_BODY	17

#define BODY_HAS_R_EYE	2
#define BODY_HAS_L_EYE	4
#define BODY_HAS_HEAD 	8
#define BODY_HAS_NECK	16
#define BODY_HAS_CHEST	32
#define BODY_HAS_BACK	64
#define BODY_HAS_R_ARM	128
#define BODY_HAS_R_HAND	256
#define BODY_HAS_L_ARM	512
#define BODY_HAS_L_HAND	1024
#define BODY_HAS_ABDOMEN 2048
#define BODY_HAS_R_LEG	4096
#define BODY_HAS_R_FOOT	8192
#define BODY_HAS_L_LEG	16384
#define	BODY_HAS_L_FOOT	32768
#define BODY_HAS_CNS	65536



#define MAX_CREATURE		91

#define MAX_GEM				50

#define MAX_ARMOR			24

#define MAX_METAL			8

#define MAX_MAGIC_ITEM		14


#define MAX_BOX_ADJ			13
#define MAX_BOX_MATL		12
#define MAX_BOX_NOUN		5

#define MAX_CONTAINER_ADJ			13
#define MAX_CONTAINER_MATL		12
#define MAX_CONTAINER_NOUN		5

/*
 * Apply types (for affects).
 * Used in #OBJECTS.
 */
#define APPLY_NONE		      0
#define APPLY_NONE1		      1
#define APPLY_NONE2		      2
#define APPLY_NONE3		      3
#define APPLY_NONE4		      4
#define APPLY_NONE5		      5
#define APPLY_SEX		      6
#define APPLY_CLASS		      7
#define APPLY_LEVEL		      8
#define APPLY_AGE		      9
#define APPLY_HEIGHT		     10
#define APPLY_WEIGHT		     11
#define APPLY_MANA		     12
#define APPLY_HIT		     13
#define APPLY_MOVE		     14
#define APPLY_UNUSED1		     15
#define APPLY_UNUSED2		     16
#define APPLY_UNUSED3		     17
#define APPLY_UNUSED4		     18
#define APPLY_UNUSED5		     19
#define APPLY_UNUSED6	     	 20
#define APPLY_UNUSED7		     21
#define APPLY_UNUSED8	     22
#define APPLY_UNUSED9     	23
#define APPLY_UNUSED10	     24
#define APPLY_STR		      25
#define APPLY_END		      26
#define APPLY_DEX		      27
#define APPLY_SPD		      28
#define APPLY_WIL		      29
#define APPLY_POT		      30
#define APPLY_JDG		      31
#define APPLY_INT		      32
#define APPLY_WIS		      33
#define APPLY_CHA		      34

#define APPLY_AS		APPLY_UNUSED1
#define APPLY_DS		APPLY_UNUSED2
#define APPLY_CS		APPLY_UNUSED3
#define APPLY_TD		APPLY_UNUSED4



/*
 * Values for containers (value[1]).
 * Used in #OBJECTS.
 */
#define CONT_CLOSEABLE		      1
#define CONT_PICKPROOF		      2
#define CONT_CLOSED		      4
#define CONT_LOCKED		      8



/*
 * Well known room virtual numbers.
 * Defined in #ROOMS.
 */
#define ROOM_VNUM_LIMBO		      2 // unused
#define ROOM_VNUM_CHAT		   	  3 // the refuge
#define ROOM_VNUM_GATE		   5004
#define ROOM_VNUM_TEMPLE	   5062
#define ROOM_VNUM_ALTAR		   5062 // temple of rebirth
#define ROOM_VNUM_YS_TEMPLE	   13013 
#define ROOM_VNUM_YS_HARBOR	   13052
#define ROOM_VNUM_SCHOOL	   12000 // unused
#define ROOM_VNUM_START			7 // select, choose, roll, rename
#define ROOM_VNUM_WEAPON		8 // pick a weapon type, or none
#define ROOM_VNUM_CITY			9 // pick Reng or Scunthorpe


#define ROOM_VNUM_WEAPON_SHOP	5068
#define ROOM_VNUM_ARMOR_SHOP	5067
#define ROOM_VNUM_WEAPON_SHOP2	13044
#define ROOM_VNUM_ARMOR_SHOP2	13029
#define ROOM_VNUM_GEMSHOP		5071
#define ROOM_VNUM_YS_GEMSHOP	13034
#define ROOM_VNUM_FURRIER		5073
#define ROOM_VNUM_YS_FURRIER	13023
#define ROOM_VNUM_MAGIC_SHOP	5070
#define ROOM_VNUM_YS_MAGIC_SHOP	13015
#define ROOM_VNUM_PAWNSHOP		5072
#define ROOM_VNUM_YS_BAR1		13019
#define ROOM_VNUM_YS_BAR2		13001
#define ROOM_VNUM_YS_BAR3		13090
#define ROOM_VNUM_YS_BAR4		13028
#define ROOM_VNUM_SCUNTHORPE_TAVERN	5075





/*
 * Room flags.
 * Used in #ROOMS.
 */
#define ROOM_DYNAMIC		      1
#define ROOM_NO_MOB		      4
#define ROOM_INDOORS		      8
#define ROOM_PRIVATE		    512
#define ROOM_SAFE		   		1024
#define ROOM_SOLITARY		   2048
#define ROOM_PET_SHOP		   4096
#define ROOM_NO_RECALL		   8192
#define ROOM_BANK		   		16384


/*
 * Directions.
 * Used in #ROOMS.
 */
#define DIR_NORTH		      0
#define DIR_EAST		      1
#define DIR_SOUTH		      2
#define DIR_WEST		      3
#define DIR_UP			      4
#define DIR_DOWN		      5
#define DIR_NORTHEAST     		6
#define DIR_SOUTHEAST    		7
#define DIR_SOUTHWEST     		8
#define DIR_NORTHWEST     		9
#define DIR_OUT				   10
#define DIR_MAX				   11



/*
 * Exit flags.
 * Used in #ROOMS.
 */
#define EX_ISDOOR		      1
#define EX_CLOSED		      2
#define EX_LOCKED		      4
#define EX_PICKPROOF		     32



/*
 * Sector types.
 * Used in #ROOMS.
 */
#define SECT_INSIDE		      0
#define SECT_CITY		      1
#define SECT_FIELD		      2
#define SECT_FOREST		      3
#define SECT_HILLS		      4
#define SECT_MOUNTAIN		      5
#define SECT_WATER_SWIM		      6
#define SECT_WATER_NOSWIM	      7
#define SECT_SWAMP	      	8
#define SECT_AIR		      9
#define SECT_DESERT		     10
#define SECT_DUNGEON		 11
#define SECT_CAVES			 12
#define SECT_PLAINS			 13
#define SECT_RUINS			 14
#define SECT_MAX		     15

#define SECT_UNDERWATER		SECT_UNUSED


#define DYNAMIC_INSIDE		      0
#define DYNAMIC_CITY		      1
#define DYNAMIC_FIELD		      2
#define DYNAMIC_FOREST		      3
#define DYNAMIC_HILLS		      4
#define DYNAMIC_MOUNTAIN		      5
#define DYNAMIC_WATER_SWIM		      6
#define DYNAMIC_WATER_NOSWIM	      7
#define DYNAMIC_UNUSED		      8
#define DYNAMIC_AIR		      9
#define DYNAMIC_DESERT		     10
#define DYNAMIC_MAX		     11




/*
 * Equpiment wear locations.
 * Used in #RESETS.
 */

#define EQUIP_NONE 			-1
#define EQUIP_RIGHTHAND 	0
#define EQUIP_LEFTHAND		1
#define EQUIP_HEAD			2
#define EQUIP_EARS			3
#define EQUIP_NECK			4
#define EQUIP_AROUND_NECK 	5
#define EQUIP_BACK			6
#define EQUIP_ARMS			7
#define EQUIP_HANDS			8
#define EQUIP_WRIST			9
#define EQUIP_FRONT			10
#define EQUIP_CHEST			11
#define EQUIP_FINGER		12
#define EQUIP_BELT			13
#define EQUIP_ONBELT		14
#define EQUIP_LEGS			15
#define EQUIP_ANKLE			16
#define EQUIP_FEET			17
#define EQUIP_PIN			18
#define EQUIP_SHOULDER		19
#define EQUIP_CLOAK			20
#define EQUIP_MAX			21


/***************************************************************************
 *                                                                         *
 *                   VALUES OF INTEREST TO AREA BUILDERS                   *
 *                   (End of this section ... stop here)                   *
 *                                                                         *
 ***************************************************************************/


#define ALIGN_NONE			0
#define ALIGN_LEFT			1
#define ALIGN_RIGHT			2
#define ALIGN_CENTER		3

#define MSL MAX_STRING_LENGTH  

/*
 * Conditions.
 */
#define COND_DRUNK		      0
#define COND_FULL		      1
#define COND_THIRST		      2



/*
 * Positions.
 */
#define POS_PRONE			      0
#define POS_SITTING				  1
#define POS_KNEELING			  2
#define POS_STANDING		      3



/*
 * ACT bits for players.
 */
#define PLR_IS_NPC		      1		/* Don't EVER set.	*/
#define PLR_CREATED		      2

#define PLR_ECHO		      8
#define PLR_RETURN	     	 16
#define PLR_COLOR            32
#define PLR_BLANK		     64
#define PLR_BRIEF		    128
#define PLR_WRAP		    512
#define PLR_PROMPT		   1024
#define PLR_TELNET_GA		   2048

#define PLR_HOLYLIGHT		   4096
#define PLR_WIZINVIS		   8192

#define	PLR_SILENCE		  32768
#define PLR_NO_EMOTE		  65536
#define PLR_NO_TELL		 262144
#define PLR_LOG			 524288
#define PLR_DENY		1048576
#define PLR_FREEZE		2097152
#define PLR_PARSE		4194304
#define PLR_COINS		8388608


/*
 * Obsolete bits.
 */
#if 0
#define PLR_AUCTION		      4	/* Obsolete	*/
#define PLR_CHAT		    256	/* Obsolete	*/
#define PLR_NO_SHOUT		 131072	/* Obsolete	*/
#endif



/*
 * Channel bits.
 */
#define	CHANNEL_AUCTION		      1
#define	CHANNEL_CHAT		      2
#define	CHANNEL_HACKER		      4
#define	CHANNEL_IMMTALK		      8
#define	CHANNEL_MUSIC		     16
#define	CHANNEL_QUESTION	     32
#define	CHANNEL_SHOUT		     64
#define	CHANNEL_YELL		    128



/*
 * Prototype for a mob.
 * This is the in-memory version of #MOBILES.
 */
struct	mob_index_data
{
    MOB_INDEX_DATA *	next;
    SPEC_FUN *		spec_fun;
    SHOP_DATA *		pShop;
	EQUIP_DATA *	equipped;
    char *		player_name;
    char *		short_descr;
    char *		long_descr;
    char *		description;
    short		vnum;
    short		count;
    short		killed;
    short		sex;
    short		level;
    int			act;
    int			affected_by;
    short		alignment;
	short		as;
	short		ds;
	short		cs;
	short		td;
	int			gold;
};



/*
 * One character (PC or NPC).
 */
struct	char_data
{
    CHAR_DATA *		next;
    CHAR_DATA *		next_in_room;
    CHAR_DATA *		master;
    CHAR_DATA *		leader;
    CHAR_DATA *		fighting;
    CHAR_DATA *		reply;
    SPEC_FUN *		spec_fun;
    MOB_INDEX_DATA *	pIndexData;
    DESCRIPTOR_DATA *	desc;
    AFFECT_DATA *	affected;
    NOTE_DATA *		pnote;
    OBJ_DATA *		carrying;
    ROOM_INDEX_DATA *	in_room;
    ROOM_INDEX_DATA *	was_in_room;
    PC_DATA *		pcdata;
	 PREDELAY_DATA *	predelay_info;
    int                 predelay_time;
    char *		name;
    char *		short_descr;
    char *		long_descr;
    char *		description;
    short		sex;
    short		class;
    short		race;
    short		level;
    short		trust;
    int			played;
    time_t		logon;
    time_t		save_time;
    short		timer;
    short		wait;
    short		hit;
    short		max_hit;
    short		mana;
    short		max_mana;
    short		move;
    short		max_move;
    int			gold;
    int			exp;
    int			act;
    int			affected_by;
    short		position;
    short		practice;
    short		carry_weight;
    short		carry_number;
    short		saving_throw;
    short		alignment;
    short		hitroll;
    short		damroll;
    short		armor;
    short		wimpy;
    short		deaf;
    short       prepared_spell;
	short		body[MAX_BODY];
	short		scars[MAX_BODY];
	short		bleed[MAX_BODY];
	short		bandage[MAX_BODY];
	short		stun;
	short		as;
	short		ds;
	short 		cs;
	short 		td;
	short		decay;
	short		stance;
	short		order;
	short		ordercost;
	CHAR_DATA * hunting;
};



/*
 * Data which only PC's have.
 */
struct	pc_data
{
    PC_DATA *		next;
    char *		pwd;
    char *		bamfin;
    char *		bamfout;
    char *		title;
	char * 		hair;
	char * 		skin;
	char *		eyes;
	short 		age;
	int			bank;			
	short		perm_Strength;
	short		perm_Endurance;
	short		perm_Dexterity;
	short		perm_Speed;
	short		perm_Willpower;
	short		perm_Potency;
	short		perm_Judgement;
	short		perm_Intelligence;
	short		perm_Wisdom;
	short		perm_Charm;
	short		mod_Strength;
	short		mod_Endurance;
	short		mod_Dexterity;
	short		mod_Speed;
	short		mod_Willpower;
	short		mod_Potency;
	short		mod_Judgement;
	short		mod_Intelligence;
	short		mod_Wisdom;
	short		mod_Charm;
    short		condition	[3];
    short		learned		[MAX_SKILL];
	short		PTPs;
	short		MTPs;
	short		spells[3];
	SKILL_DELAY_DATA *	skill_delays;
	int			fame;
	short		deeds;
	short		wrap;
	short		fieldExp;			// not implemented
};





/*
 * Extra description data for a room or object.
 */
struct	extra_descr_data
{
    EXTRA_DESCR_DATA *next;	/* Next in list                     */
    char *keyword;              /* Keyword in look/examine          */
    char *description;          /* What to see                      */
};

/*
 * Verb trap data for an obj
 */
struct	verb_trap_data
{
    VERB_TRAP_DATA *next;	/* Next in list                     */
    char *verb;              /* what verb, could be real or just for this obj */
    char *firstPersonMessage;          // to_char
	char *roomMessage;	// to_room
	char *GSL;
};





/*
 * Prototype for an object.
 */
struct	obj_index_data
{
    OBJ_INDEX_DATA *	next;
    EXTRA_DESCR_DATA *	extra_descr;
    AFFECT_DATA *	affected;
	VERB_TRAP_DATA *	verb_trap;
    char *		name;
    char *		short_descr;
    char *		description;
    short		vnum;
    short		item_type;
    long		extra_flags;
    long		wear_flags;
    short		count;
    short		weight;
    int			cost;
	short		st;
	short		du;
    int			value	[4];
	char		* obj_spec_name;
	OBJ_FUN	*	spec_fun;
};



/*
 * One object.
 */
struct	obj_data
{
    OBJ_DATA *		next;
    OBJ_DATA *		next_content;
    OBJ_DATA *		contains;
    OBJ_DATA *		in_obj;
	OBJ_FUN *		spec_fun;
    CHAR_DATA *		carried_by;
    EXTRA_DESCR_DATA *	extra_descr;
    AFFECT_DATA *	affected;
    OBJ_INDEX_DATA *	pIndexData;
    ROOM_INDEX_DATA *	in_room;
    char *		name;
    char *		short_descr;
    char *		description;
    short		item_type;
    short		extra_flags;
    long		wear_flags;
    short		wear_loc;
    short		weight;
    int			cost;
	short		st;
	short		du;
    short		level;
    short		timer;
    int			value	[4];
	char *		obj_spec_name;
	VERB_TRAP_DATA *verb_trap;
};



/*
 * Exit data.
 */
struct	exit_data
{
    ROOM_INDEX_DATA *	to_room;
    short		vnum;
    short		exit_info;
    short		key;
    char *		keyword;
    char *		description;
};



/*
 * Reset commands:
 *   '*': comment
 *   'M': read a mobile
 *   'O': read an object
 *   'P': put object in object
 *   'G': give object to mobile
 *   'E': equip object to mobile
 *   'D': set state of door
 *   'R': randomize room exits
 *   'S': stop (end of list)
 */
 
struct	equip_data
{
    EQUIP_DATA *	next;
    int			item;
    int			location;
};

/*
 * Area-reset definition.
 */
struct	reset_data
{
    RESET_DATA *	next;
    char		command;
    short		arg1;
    short		arg2;
    short		arg3;
};



/*
 * Area definition.
 */
struct	area_data
{
    AREA_DATA *		next;
    RESET_DATA *	reset_first;
    RESET_DATA *	reset_last;
	ROOM_INDEX_DATA *	room_first;
    ROOM_INDEX_DATA *	room_last;
    char *		name;
    short		age;
    short		nplayer;
	short		ncritter;
	char *desc[10];
	char *builders;
	char *filename;
    int			vnum_start;
    int			vnum_final;
    int			reset_length;
    int			area_bits;
	int	creatures[5];
};



/*
 * Room type.
 */
struct	room_index_data
{
    ROOM_INDEX_DATA *	next;
    ROOM_INDEX_DATA *	zone_next;
    CHAR_DATA *		people;
    OBJ_DATA *		contents;
    EXTRA_DESCR_DATA *	extra_descr;
    AREA_DATA *		area;
    EXIT_DATA *		exit	[11];
	ROOM_SPECIAL *	spec_fun;
    char *		name;
    char *		description;
    short		vnum;
    short		room_flags;
    short		light;
    short		sector_type;
};



/*
 * Types of attacks.
 * Must be non-overlapping with spell/skill types,
 * but may be arbitrary beyond that.
 */
#define TYPE_UNDEFINED               -1
#define TYPE_HIT                     1000
#define TYPE_BOLT_SPELL				 2000
#define TYPE_OBJECT_DAMAGE			 500 // traps, shrapnel, etc.
#define TYPE_BLEED					 750

// damage types

#define DAMAGE_TYPE_NONSPECIFIC			0
#define DAMAGE_TYPE_PUNCTURE			1
#define DAMAGE_TYPE_CRUSH				2
#define DAMAGE_TYPE_SLASH				3
#define DAMAGE_TYPE_SHOCK				4
#define DAMAGE_TYPE_WATER				5
#define DAMAGE_TYPE_ACID				6
#define DAMAGE_TYPE_FIRE				7
#define DAMAGE_TYPE_COLD				8
#define DAMAGE_TYPE_DISINTEGRATE		9
#define DAMAGE_TYPE_OBJECT				10   //not used..
#define DAMAGE_TYPE_BLEED				11   //not used

// crit table

#define MAX_CRIT 				10

#define STATUS_NONE				0
#define STATUS_STUN_1			1
#define	STATUS_STUN_2			2
#define STATUS_STUN_3			4
#define STATUS_STUN_4			8
#define	STATUS_STUN_5			16
#define STATUS_STUN_6			32
#define STATUS_STUN_7			64
#define	STATUS_STUN_8			128
#define STATUS_STUN_9			256
#define STATUS_STUN_10			512
#define STATUS_STUN_11			1024
#define STATUS_STUN_12			2048
#define STATUS_KNOCKDOWN		4096
#define STATUS_DEATH			8192

#define WOUND_NONE				0
#define WOUND_RANK_1			1
#define WOUND_RANK_2			2
#define WOUND_RANK_3			3

#define BLEED_NONE				0
#define BLEED_1PER				1
#define BLEED_2PER				2
#define BLEED_3PER				3
#define BLEED_4PER				4
#define BLEED_6PER				6
#define BLEED_8PER				8
#define BLEED_10PER			   10

/*
 *  Target types.
 */
#define TAR_IGNORE		    0
#define TAR_CHAR_OFFENSIVE	    1
#define TAR_CHAR_DEFENSIVE	    2
#define TAR_CHAR_SELF		    3
#define TAR_OBJ_INV		    4
#define TAR_EITHER			5
#define TAR_CHAR_HEALING	6


#define SKILL_NOT_IMPLEMENTED	-1
#define SKILL_GENERAL			0
#define SKILL_COMBAT			1
#define SKILL_MAGIC				2
#define SKILL_STEALTH			3

#define DIFFICULTY_TRIVIAL	1
#define DIFFICULTY_EASY		2
#define DIFFICULTY_NORMAL	3
#define DIFFICULTY_HARD		4
#define DIFFICULTY_ARDUOUS	5

#define STAT_STRENGTH		1
#define STAT_ENDURANCE		2
#define STAT_DEXTERITY		3
#define STAT_SPEED			4
#define STAT_WILLPOWER		5
#define STAT_POTENCY		6
#define STAT_JUDGEMENT		7
#define STAT_INTELLIGENCE	8
#define STAT_WISDOM			9
#define STAT_CHARM			10


//blah blah poorly organized blah blah

#define BYPASS_STUN	1
#define BYPASS_RT	2
#define BYPASS_DEAD 4
#define CMD_DONT_PARSE 8
#define COMMAND_OLC 16
#define CMD_BREAKS_HIDE 32
#define CMD_BREAKS_PREDELAY 64


/*
 * Command logging types.
 */
#define LOG_NORMAL	0
#define LOG_ALWAYS	1
#define LOG_NEVER	2


/*
 * Skills include spells as a particular case.
 */
struct	skill_type
{
    char *	name;			/* Name of skill		*/
	short	type;			// what type of skill is it
	short	difficulty;		// difficulty modifier
	short	basePTPs;		// base PTP cost 1 train
	short	baseMTPs;		// base MTP cost 1 train
	short	stat;			// governed by what stat
	short *	pgsn;			/* Pointer to associated gsn	*/
};

struct	spell_type
{
	short   prof; /* What profession uses this spell */
	short	number;
    char *	name;			/* Name of skill		*/
	short level;
    SPELL_FUN *	spell_fun;		/* Spell pointer (for spells)	*/
    short	target;			/* Legal targets		*/
    short	minimum_position;	/* Position for caster / user	*/
	short *	pgsn;			/* Pointer to associated gsn	*/
    short	slot;			/* Slot for #OBJECT loading	*/
    short	min_mana;		/* Minimum mana used		UNUSED */
    short	beats;			/* Waiting time after use	UNUSED */
    char *	msg_off;		/* Wear off message		*/
	char *	msg_off2;		/* Wear off message		*/
};


/*
 * These are skill_lookup return values for common skills and spells.
 */
extern	short	gsn_armor_use;
extern	short	gsn_shield_use;

extern	short	gsn_edged_weapons;
extern	short	gsn_blunt_weapons;
extern	short	gsn_brawling;
extern	short	gsn_two_handed_weapons;
extern	short	gsn_ranged_weapons;
extern	short	gsn_thrown_weapons;
extern	short	gsn_polearm_weapons;

extern	short	gsn_ambush;
extern	short	gsn_two_weapon_combat;
extern	short	gsn_combat_maneuvers;
extern	short	gsn_multi_opponent_combat;
extern	short	gsn_physical_fitness;
extern	short	gsn_dodging;
extern	short	gsn_arcane_symbols;
extern	short	gsn_magic_item_use;
extern	short	gsn_spell_aiming;
extern	short	gsn_harness_power;
extern	short	gsn_elemental_mana_control;
extern	short	gsn_mental_mana_control;
extern	short	gsn_spirit_mana_control;
extern	short	gsn_spell_research;
extern	short	gsn_elemental_lore;
extern	short	gsn_spiritual_lore;
extern	short	gsn_sorcerous_lore;
extern	short	gsn_mental_lore;
extern	short	gsn_survival;
extern	short	gsn_disarming_traps;
extern	short	gsn_picking_locks;
extern	short	gsn_stalking_and_hiding;
extern	short	gsn_perception;
extern	short	gsn_climbing;
extern	short	gsn_swimming;
extern	short	gsn_first_aid;
extern	short	gsn_trading;
extern	short	gsn_pickpocketing;


extern	short	gsn_elemental_defense_i;
extern	short	gsn_elemental_defense_ii;
extern	short	gsn_elemental_defense_iii;
extern	short	gsn_spirit_warding_i;
extern	short	gsn_spirit_warding_ii;
extern	short	gsn_spirit_defense;
extern	short	gsn_spirit_barrier;
extern	short	gsn_spirit_shield;
extern	short	gsn_spell_shield;
extern	short	gsn_lesser_shroud;
extern	short	gsn_protective_ward;
extern	short	gsn_strength;
extern	short	gsn_heal_i;
extern	short	gsn_heal_ii;
extern	short	gsn_heal_iii;
extern	short	gsn_heal_scars_i;
extern	short	gsn_heal_scars_ii;
extern	short	gsn_heal_scars_iii;
extern	short	gsn_restore_health_i;
extern	short	gsn_restore_health_ii;
extern  short	gsn_resurrect;
extern	short	gsn_spirit_guide;
extern	short	gsn_transference;
extern	short	gsn_animate_corpse;
extern	short	gsn_bleed;
extern	short	gsn_disrupt;
extern	short	gsn_maelstrom;
extern	short	gsn_curse;
extern	short	gsn_summon_demon;
extern	short	gsn_sorcerous_travel;
extern short	gsn_sleep;
extern short 	gsn_bless_weapon;
extern	short	gsn_invis;

extern	short	gsn_hold_undead;
extern	short	gsn_destroy_undead;
extern  short   gsn_death_ray;

/*
 * Utility macros.
 */
#define UMIN(a, b)		((a) < (b) ? (a) : (b))
#define UMAX(a, b)		((a) > (b) ? (a) : (b))
#define URANGE(a, b, c)		((b) < (a) ? (a) : ((b) > (c) ? (c) : (b)))
#define LOWER(c)		((c) >= 'A' && (c) <= 'Z' ? (c)+'a'-'A' : (c))
#define UPPER(c)		((c) >= 'a' && (c) <= 'z' ? (c)+'A'-'a' : (c))
#define IS_SET(flag, bit)	((flag) & (bit))
#define SET_BIT(var, bit)	((var) |= (bit))
#define REMOVE_BIT(var, bit)	((var) &= ~(bit))



/*
 * Character macros.
 */
#define IS_DEAD(ch) (IS_SET( (ch)->affected_by, AFF_DEAD ) || (ch)->hit < 1 )
 
#define IS_NPC(ch)		(IS_SET((ch)->act, ACT_IS_NPC))
#define IS_IMMORTAL(ch)		(get_trust(ch) >= LEVEL_IMMORTAL)
#define IS_HERO(ch)		(get_trust(ch) >= LEVEL_HERO)
#define IS_AFFECTED(ch, sn)	(IS_SET((ch)->affected_by, (sn)))

#define IS_NEWBIE(ch) (!IS_NPC(ch) && get_hours_played(ch) == 0)

#define IS_AWAKE(ch)		(!IS_SET((ch)->affected_by, AFF_SLEEP))

#define IS_OUTSIDE(ch)		(!IS_SET(				    \
				    (ch)->in_room->room_flags,		    \
				    ROOM_INDOORS) &&((ch)->in_room->sector_type != SECT_INSIDE && ( (ch)->in_room->sector_type != SECT_INSIDE && (ch)->in_room->sector_type != SECT_DUNGEON && (ch)->in_room->sector_type != SECT_CAVES ) ) )

#define IS_WEAPON(dt) ((dt) > TYPE_HIT && (dt) <=  (TYPE_HIT+MAX_REAL_WEAPON) )
#define IS_MELEE(dt) ((dt) >= TYPE_HIT && (dt) < TYPE_BOLT_SPELL)
#define IS_BOLT(dt) ((dt) >= TYPE_BOLT_SPELL)
#define IS_OBJECT(dt) ((dt) == TYPE_OBJECT_DAMAGE )
#define IS_BLEED(dt) ((dt) == TYPE_BLEED )

/*
 * Object macros.
 */
#define CAN_WEAR(obj, part)	(IS_SET((obj)->wear_flags,  (part)))
#define IS_OBJ_STAT(obj, stat)	(IS_SET((obj)->extra_flags, (stat)))



/*
 * Description macros.
 */
 
/*
#define PERS(ch, looker)	( can_see( looker, (ch) ) ?		\
				( IS_NPC(ch) ? (ch)->short_descr	\
				: (ch)->name ) : "someone" )
*/

#define PERS(ch, looker) pers((ch), (looker))
#define CAP_PERS(ch, looker) cap_pers((ch), (looker))


//added

#define STR(dat, field)         (( (dat)->field != NULL                    \
					     ? (dat)->field                \
					     : (dat)->pIndexData->field ))
#define NST(pointer)            (pointer == NULL ? "" : pointer)

/*
 * Articles.
 */
#define HE_SHE(ch)           (((ch)->sex == SEX_MALE  ) ? "he"   :  \
			   (  ((ch)->sex == SEX_FEMALE) ? "she"  : "it" ) )
#define HIS_HER(ch)          (((ch)->sex == SEX_MALE  ) ? "his"  :  \
			   (  ((ch)->sex == SEX_FEMALE) ? "her"  : "its" ) )
#define HIM_HER(ch)          (((ch)->sex == SEX_MALE  ) ? "him"  :  \
			   (  ((ch)->sex == SEX_FEMALE) ? "her"  : "it" ) )



/*
 * Structure for a command in the command lookup table.
 */
struct	cmd_type
{
    char * const	name;
    DO_FUN *		do_fun;
    short		position;
    short		level;
    short		log;
	int			flags;
};



/*
 * Structure for a social in the socials table.
 */
struct	social_type
{
    char * const	name;
    char * const	char_no_arg;
    char * const	others_no_arg;
    char * const	char_found;
    char * const	others_found;
    char * const	vict_found;
    char * const	char_auto;
    char * const	others_auto;
	char * const	obj_inv_self; // socials that affect objects just like in GS!
    char * const	obj_inv_room;
	char * const	obj_room_self;
    char * const	obj_room_room;
};



/*
 * Global constants.
 */

extern	const	struct	class_type	class_table	[MAX_CLASS];
extern	const	struct	race_type	race_table	[MAX_RACE];
extern	const	struct	cmd_type	cmd_table	[];


extern	const	struct	skill_type	skill_table	[MAX_SKILL];
extern	const	struct	spell_type	spell_table	[MAX_SPELL];
extern	const	struct	social_type	social_table	[];

extern	char *	const			body_name	[MAX_BODY];

//only needed in track.c
//extern char *	const	dir_name[MAX_DIRECTION];
//extern char *	const	rev_dir[MAX_DIRECTION];


extern const struct box_matl_type box_matl_table [MAX_BOX_MATL];
extern const struct box_adj_type box_adj_table [MAX_BOX_ADJ];
extern const struct box_noun_type box_noun_table [MAX_BOX_NOUN];

extern const struct container_matl_type container_matl_table [MAX_CONTAINER_MATL];
extern const struct container_adj_type container_adj_table [MAX_CONTAINER_ADJ];
extern const struct container_noun_type container_noun_table [MAX_CONTAINER_NOUN];

extern const struct magic_item_type magic_item_table [MAX_MAGIC_ITEM];
extern const struct metal_type metal_table [MAX_METAL];
extern const struct armor_type armor_table [MAX_ARMOR];
extern const struct gem_type gem_table [MAX_GEM];

extern const struct creature_type creature_table [MAX_CREATURE];

extern const struct weapon_type weapon_table [MAX_WEAPON];
extern const struct bolt_type bolt_table [MAX_BOLT_SPELL];
extern const struct crit_type crit_table [MAX_BODY][MAX_CRIT];

extern const struct shield_type shield_table [MAX_SHIELD];

extern const struct quest_type faction_table [MAX_FACTION];
extern const struct quest_type quest_table [MAX_QUEST];


/*
 * Global variables.
 */
extern		HELP_DATA	  *	help_first;
extern		SHOP_DATA	  *	shop_first;

extern		BAN_DATA	  *	ban_list;
extern		CHAR_DATA	  *	char_list;
extern		DESCRIPTOR_DATA   *	descriptor_list;
extern		NOTE_DATA	  *	note_list;
extern		OBJ_DATA	  *	object_list;

extern		AFFECT_DATA	  *	affect_free;
extern		BAN_DATA	  *	ban_free;
extern		CHAR_DATA	  *	char_free;
extern		DESCRIPTOR_DATA	  *	descriptor_free;
extern		EXTRA_DESCR_DATA  *	extra_descr_free;
extern		VERB_TRAP_DATA  *	verb_trap_free;
extern		NOTE_DATA	  *	note_free;
extern		OBJ_DATA	  *	obj_free;
extern		PC_DATA		  *	pcdata_free;
extern		QUEST_DATA    * quest_free;

extern		char			bug_buf		[];
extern		time_t			current_time;
extern		bool			fLogAll;
extern		FILE *			fpReserve;
extern		KILL_DATA		kill_table	[];
extern		char			log_buf		[];
extern		TIME_INFO_DATA		time_info;
extern		WEATHER_DATA		weather_info;
extern		SKILL_DELAY_DATA  *	skill_delay_free;



/*
 * Command functions.
 * Defined in act_*.c (mostly).
 */
DECLARE_DO_FUN(	do_advance	);
DECLARE_DO_FUN(	do_allow	);
DECLARE_DO_FUN(	do_ambush	);
DECLARE_DO_FUN(	do_answer	);
DECLARE_DO_FUN(	do_areas	);
DECLARE_DO_FUN(	do_at		);
DECLARE_DO_FUN(	do_auction	);
DECLARE_DO_FUN(	do_backstab	);
DECLARE_DO_FUN(	do_bamfin	);
DECLARE_DO_FUN(	do_bamfout	);
DECLARE_DO_FUN(	do_ban		);
DECLARE_DO_FUN(	do_bug		);
DECLARE_DO_FUN(	do_buy		);
DECLARE_DO_FUN(	do_cast		);
DECLARE_DO_FUN(	do_channels	);
DECLARE_DO_FUN(	do_chat		);
DECLARE_DO_FUN(	do_close	);
DECLARE_DO_FUN(	do_commands	);
DECLARE_DO_FUN(	do_compare	);
DECLARE_DO_FUN(	do_config	);
DECLARE_DO_FUN(	do_consider	);
DECLARE_DO_FUN(	do_credits	);
DECLARE_DO_FUN(	do_deny		);
DECLARE_DO_FUN(	do_disarm	);
DECLARE_DO_FUN(	do_disconnect	);
DECLARE_DO_FUN(	do_down		);
DECLARE_DO_FUN(	do_drink	);
DECLARE_DO_FUN(	do_drop		);
DECLARE_DO_FUN(	do_east		);
DECLARE_DO_FUN(	do_eat		);
DECLARE_DO_FUN(	do_echo		);
DECLARE_DO_FUN(	do_emote	);
DECLARE_DO_FUN(	do_equipment	);
DECLARE_DO_FUN(	do_examine	);
DECLARE_DO_FUN(	do_exits	);
DECLARE_DO_FUN(	do_fill		);
DECLARE_DO_FUN(	do_flee		);
DECLARE_DO_FUN(	do_follow	);
DECLARE_DO_FUN(	do_force	);
DECLARE_DO_FUN(	do_freeze	);
DECLARE_DO_FUN(	do_get		);
DECLARE_DO_FUN(	do_glance );
DECLARE_DO_FUN(	do_give		);
DECLARE_DO_FUN(	do_goto		);
DECLARE_DO_FUN(	do_group	);
DECLARE_DO_FUN(	do_gtell	);
DECLARE_DO_FUN(	do_help		);
DECLARE_DO_FUN(	do_hide		);
DECLARE_DO_FUN(	do_holylight	);
DECLARE_DO_FUN(	do_idea		);
DECLARE_DO_FUN(	do_immtalk	);
DECLARE_DO_FUN(	do_inventory	);
DECLARE_DO_FUN(	do_invis	);
DECLARE_DO_FUN(	do_kick		);
DECLARE_DO_FUN(	do_kill		);
DECLARE_DO_FUN(	do_list		);
DECLARE_DO_FUN(	do_lock		);
DECLARE_DO_FUN(	do_log		);
DECLARE_DO_FUN(	do_look		);
DECLARE_DO_FUN(	do_memory	);
DECLARE_DO_FUN(	do_mfind	);
DECLARE_DO_FUN(	do_mload	);
DECLARE_DO_FUN(	do_mset		);
DECLARE_DO_FUN(	do_mstat	);
DECLARE_DO_FUN(	do_mwhere	);
DECLARE_DO_FUN(	do_murde	);
DECLARE_DO_FUN(	do_murder	);
DECLARE_DO_FUN(	do_music	);
DECLARE_DO_FUN(	do_noemote	);
DECLARE_DO_FUN(	do_north	);
DECLARE_DO_FUN(	do_note		);
DECLARE_DO_FUN(	do_notell	);
DECLARE_DO_FUN(	do_ofind	);
DECLARE_DO_FUN(	do_oload	);
DECLARE_DO_FUN(	do_open		);
DECLARE_DO_FUN(	do_order	);
DECLARE_DO_FUN(	do_oset		);
DECLARE_DO_FUN(	do_ostat	);
DECLARE_DO_FUN(	do_pardon	);
DECLARE_DO_FUN(	do_password	);
DECLARE_DO_FUN(	do_peace	);
DECLARE_DO_FUN(	do_pick		);
DECLARE_DO_FUN(	do_pose		);
DECLARE_DO_FUN(	do_practice	);
DECLARE_DO_FUN(	do_prepare	);
DECLARE_DO_FUN(	do_purge	);
DECLARE_DO_FUN(	do_put		);
DECLARE_DO_FUN(	do_newput		);
DECLARE_DO_FUN(	do_quaff	);
DECLARE_DO_FUN(	do_question	);
DECLARE_DO_FUN(	do_qui		);
DECLARE_DO_FUN(	do_quit		);
DECLARE_DO_FUN(	do_reboo	);
DECLARE_DO_FUN(	do_reboot	);
DECLARE_DO_FUN(	do_recall	);
DECLARE_DO_FUN(	do_recho	);
DECLARE_DO_FUN(	do_recite	);
DECLARE_DO_FUN(	do_release	);
DECLARE_DO_FUN(	do_remove	);
DECLARE_DO_FUN(	do_rent		);
DECLARE_DO_FUN(	do_reply	);
DECLARE_DO_FUN(	do_report	);
DECLARE_DO_FUN(	do_rescue	);
DECLARE_DO_FUN(	do_rest		);
DECLARE_DO_FUN(	do_restore	);
DECLARE_DO_FUN(	do_return	);
DECLARE_DO_FUN(	do_rset		);
DECLARE_DO_FUN(	do_rstat	);
DECLARE_DO_FUN(	do_rub		);
DECLARE_DO_FUN(	do_sacrifice	);
DECLARE_DO_FUN(	do_save		);
DECLARE_DO_FUN(	do_say		);
DECLARE_DO_FUN(	do_score	);
DECLARE_DO_FUN(	do_score2	);
DECLARE_DO_FUN(	do_sell		);
DECLARE_DO_FUN(	do_shout	);
DECLARE_DO_FUN(	do_show		);
DECLARE_DO_FUN(	do_shutdow	);
DECLARE_DO_FUN(	do_shutdown	);
DECLARE_DO_FUN(	do_silence	);
DECLARE_DO_FUN(	do_sla		);
DECLARE_DO_FUN(	do_slay		);
DECLARE_DO_FUN(	do_sleep	);
DECLARE_DO_FUN(	do_slookup	);
DECLARE_DO_FUN(	do_sneak	);
DECLARE_DO_FUN(	do_snoop	);
DECLARE_DO_FUN(	do_socials	);
DECLARE_DO_FUN(	do_south	);
DECLARE_DO_FUN(	do_split	);
DECLARE_DO_FUN(	do_sset		);
DECLARE_DO_FUN(	do_stand	);
DECLARE_DO_FUN(	do_steal	);
DECLARE_DO_FUN(	do_switch	);
DECLARE_DO_FUN(	do_tell		);
DECLARE_DO_FUN(	do_tend		);
DECLARE_DO_FUN(	do_time		);
DECLARE_DO_FUN(	do_skills	);
DECLARE_DO_FUN(	do_transfer	);
DECLARE_DO_FUN(	do_trust	);
DECLARE_DO_FUN(	do_typo		);
DECLARE_DO_FUN(	do_unlock	);
DECLARE_DO_FUN(	do_up		);
DECLARE_DO_FUN(	do_users	);
DECLARE_DO_FUN(	do_value	);
DECLARE_DO_FUN(	do_wave		);
DECLARE_DO_FUN(	do_wake		);
DECLARE_DO_FUN(	do_wear		);
DECLARE_DO_FUN(	do_weather	);
DECLARE_DO_FUN(	do_west		);
DECLARE_DO_FUN(	do_where	);
DECLARE_DO_FUN(	do_who		);
DECLARE_DO_FUN(	do_wimpy	);
DECLARE_DO_FUN(	do_wrap		);
DECLARE_DO_FUN(	do_wizhelp	);
DECLARE_DO_FUN(	do_wizlock	);
DECLARE_DO_FUN(	do_yell		);

DECLARE_DO_FUN(	do_body		);

DECLARE_DO_FUN(	do_northeast	);
DECLARE_DO_FUN(	do_northwest	);
DECLARE_DO_FUN(	do_southeast	);
DECLARE_DO_FUN(	do_southwest	);
DECLARE_DO_FUN(	do_out      	);

DECLARE_DO_FUN(	do_enter     	);
DECLARE_DO_FUN(	do_health    	);
DECLARE_DO_FUN(	do_swap      	);

DECLARE_DO_FUN(	do_kneel	);
DECLARE_DO_FUN(	do_sit	);
DECLARE_DO_FUN(	do_lie		);

DECLARE_DO_FUN(	do_experience	);
DECLARE_DO_FUN(	do_wealth	);
DECLARE_DO_FUN(	do_mana	);
DECLARE_DO_FUN(	do_spells	);
DECLARE_DO_FUN(	do_stance	);

DECLARE_DO_FUN(	do_cload	);
DECLARE_DO_FUN(	do_rload	);

DECLARE_DO_FUN(	do_search	);
DECLARE_DO_FUN(	do_search_room	);
DECLARE_DO_FUN(	do_depart	);

DECLARE_DO_FUN(	do_parse	);
DECLARE_DO_FUN(	do_hset		);
DECLARE_DO_FUN(	do_newhelp	);
DECLARE_DO_FUN(	do_savehelps);
DECLARE_DO_FUN(	do_savezone	);
DECLARE_DO_FUN(	do_break	);
DECLARE_DO_FUN(	do_collaps	);
DECLARE_DO_FUN(	do_collapse	);
DECLARE_DO_FUN(	do_connect	);
DECLARE_DO_FUN(	do_dig		);
DECLARE_DO_FUN(	do_exset	);
DECLARE_DO_FUN(	do_rdesc	);
DECLARE_DO_FUN(	do_ddesc	);
DECLARE_DO_FUN(	do_rextra	);

DECLARE_DO_FUN(	do_order2	);
DECLARE_DO_FUN(	do_buy2	);
DECLARE_DO_FUN(	do_sell2		);
DECLARE_DO_FUN(	do_appraise		);
DECLARE_DO_FUN(	do_exchange	);           //fixme

DECLARE_DO_FUN(	do_randarmor		);
DECLARE_DO_FUN(	do_randweapon	);
DECLARE_DO_FUN(	do_randbox	);
DECLARE_DO_FUN(	do_randmagic	);
DECLARE_DO_FUN(	do_randshield		);
DECLARE_DO_FUN(	do_randcontainer	);
DECLARE_DO_FUN(	do_randgem	);

DECLARE_DO_FUN(	do_unhide	);
DECLARE_DO_FUN(	do_ambush	);
DECLARE_DO_FUN(	do_search	);
DECLARE_DO_FUN(	do_point	);

DECLARE_DO_FUN(	do_disarm	);

DECLARE_DO_FUN(	do_age	);
DECLARE_DO_FUN(	do_eyes	);
DECLARE_DO_FUN(	do_skincolor	);
DECLARE_DO_FUN(	do_hair	);

DECLARE_DO_FUN(	do_select );
DECLARE_DO_FUN(	do_choose );

DECLARE_DO_FUN(	do_begin );
DECLARE_DO_FUN(	do_roll );
DECLARE_DO_FUN(	do_rename );

DECLARE_DO_FUN(	do_balance );
DECLARE_DO_FUN(	do_deposit );
DECLARE_DO_FUN(	do_withdraw );

DECLARE_DO_FUN(	do_peer );
DECLARE_DO_FUN(	do_weigh );
DECLARE_DO_FUN(	do_skin	);

DECLARE_DO_FUN(	do_exi		);
DECLARE_DO_FUN(	do_exit		);

DECLARE_DO_FUN(	do_read		);

DECLARE_DO_FUN(	do_loot		);

DECLARE_DO_FUN(	do_viewskills		);

DECLARE_DO_FUN(	do_weight		);
DECLARE_DO_FUN(	do_encumbrance		);

DECLARE_DO_FUN( do_drag );

DECLARE_DO_FUN( do_invoke );
DECLARE_DO_FUN( do_knock );

DECLARE_DO_FUN(	do_faction	);
DECLARE_DO_FUN(	do_change		);


/*
 * Spell functions.
 * Defined in magic.c.
 */
DECLARE_SPELL_FUN(	spell_null		);

DECLARE_SPELL_FUN(	spell_spirit_warding_i	);
DECLARE_SPELL_FUN(	spell_spirit_warding_ii	);
DECLARE_SPELL_FUN(	spell_heal_iii	);
DECLARE_SPELL_FUN(	spell_heal_ii	);
DECLARE_SPELL_FUN(	spell_heal_i	);
DECLARE_SPELL_FUN(	spell_restore_health_i	);
DECLARE_SPELL_FUN(	spell_heal_scars_iii	);
DECLARE_SPELL_FUN(	spell_heal_scars_ii	);
DECLARE_SPELL_FUN(	spell_heal_scars_i	);
DECLARE_SPELL_FUN(	spell_restore_health_ii	);
DECLARE_SPELL_FUN(	spell_resurrect	);
DECLARE_SPELL_FUN(	spell_unstun		);
DECLARE_SPELL_FUN(	spell_spirit_defense	);
DECLARE_SPELL_FUN(	spell_spirit_shield	);
DECLARE_SPELL_FUN(	spell_spirit_barrier	);
DECLARE_SPELL_FUN(	spell_spell_shield );
DECLARE_SPELL_FUN(	spell_lesser_shroud	);
DECLARE_SPELL_FUN(	spell_spirit_guide );
DECLARE_SPELL_FUN(	spell_transference	);

DECLARE_SPELL_FUN(	spell_holy_bolt );

DECLARE_SPELL_FUN(	spell_elemental_defense_i	);
DECLARE_SPELL_FUN(	spell_elemental_defense_ii	);
DECLARE_SPELL_FUN(	spell_elemental_defense_iii	);
DECLARE_SPELL_FUN(	spell_strength );
DECLARE_SPELL_FUN(	spell_protective_ward );
DECLARE_SPELL_FUN(	spell_sleep );
DECLARE_SPELL_FUN(	spell_shock_bolt );
DECLARE_SPELL_FUN(	spell_fire_bolt	);
DECLARE_SPELL_FUN(	spell_water_bolt );
DECLARE_SPELL_FUN(	spell_lightning_bolt );
DECLARE_SPELL_FUN(	spell_fireball		);
DECLARE_SPELL_FUN(	spell_cone_of_lightning		);
DECLARE_SPELL_FUN(	spell_identify );
DECLARE_SPELL_FUN(	spell_steam		);
DECLARE_SPELL_FUN(	spell_stone		);

DECLARE_SPELL_FUN(	spell_harm		);
DECLARE_SPELL_FUN(	spell_jolt		);
DECLARE_SPELL_FUN(	spell_bleed );
DECLARE_SPELL_FUN(	spell_disrupt	);
DECLARE_SPELL_FUN(	spell_maelstrom );
DECLARE_SPELL_FUN(	spell_curse );
DECLARE_SPELL_FUN(	spell_animate_corpse	);
DECLARE_SPELL_FUN(	spell_summon_demon );
DECLARE_SPELL_FUN(	spell_sorcerous_travel );

DECLARE_SPELL_FUN(	spell_bless_weapon );
DECLARE_SPELL_FUN(	spell_invis );
DECLARE_SPELL_FUN(	spell_enchant );
DECLARE_SPELL_FUN(	spell_summon_familiar );

DECLARE_SPELL_FUN(	spell_hold_undead );
DECLARE_SPELL_FUN(	spell_destroy_undead );
DECLARE_SPELL_FUN(	spell_death_ray );

DECLARE_SPELL_FUN(	spell_mark );
DECLARE_SPELL_FUN(	spell_conjure );



char *	crypt		args( ( const char *key, const char *salt ) );

/*
 * Data files used by the server.
 *
 * AREA_LIST contains a list of areas to boot.
 * All files are read in completely at bootup.
 * Most output files (bug, idea, typo, shutdown) are append-only.
 *
 * The NULL_FILE is held open so that we have a stream handle in reserve,
 *   so players can go ahead and telnet to all the other descriptors.
 * Then we close it whenever we need to open a file (e.g. a save file).
 */

#define PLAYER_DIR	"../player/"	/* Player files			*/
#define NULL_FILE	"/dev/null"	/* To reserve one stream	*/

#define AREA_LIST	"area.lst"	/* List of areas		*/

#define BUG_FILE	"bugs.txt"      /* For bug( ) and 'bug' and 'typo' and 'idea' */

#define NOTE_FILE   "notes.txt"

#define SHUTDOWN_FILE	"shutdown.txt"	/* For 'shutdown'		*/



/*
 * Our function prototypes.
 * One big lump ... this is every function in Merc.
 */
#define CD	CHAR_DATA
#define MID	MOB_INDEX_DATA
#define OD	OBJ_DATA
#define OID	OBJ_INDEX_DATA
#define RID	ROOM_INDEX_DATA
#define SF	SPEC_FUN

/* act_comm.c */
void	add_follower	args( ( CHAR_DATA *ch, CHAR_DATA *master ) );
void	stop_follower	args( ( CHAR_DATA *ch ) );
void	die_follower	args( ( CHAR_DATA *ch ) );
bool	is_same_group	args( ( CHAR_DATA *ach, CHAR_DATA *bch ) );

/* act_info.c */
bool new2_show_list_to_char args( ( OBJ_DATA *list, CHAR_DATA *ch, bool fShort,
  bool fShowNothing, char *pre ) );
char*	pers		args( ( CHAR_DATA *ch, CHAR_DATA *looker ) );
char*	cap_pers		args( ( CHAR_DATA *ch, CHAR_DATA *looker ) );
  
char * format_str_len args( (char * string, int length, int align)) ;
char *format_string args((char *oldstring, CHAR_DATA *ch));
void show_hands_to_char args( ( CHAR_DATA *victim, CHAR_DATA *ch ) );
int get_encumbrance_level args( ( CHAR_DATA *ch ) );

/* act_move.c */
void	move_char	args( ( CHAR_DATA *ch, int door ) );

/* act_obj.c */
int 	get_gsn args( ( int number ) );
void	get_obj		args( ( CHAR_DATA *ch, OBJ_DATA *obj,
			    OBJ_DATA *container ) );
short get_creature_number	args( ( const char *name ) );
				
/* act_wiz.c */

/* comm.c */
void	close_socket	args( ( DESCRIPTOR_DATA *dclose ) );
void	write_to_buffer	args( ( DESCRIPTOR_DATA *d, const char *txt,
			    int length ) );
void	send_to_char	args( ( const char *txt, CHAR_DATA *ch ) );
void	act		args( ( const char *format, CHAR_DATA *ch,
			    const void *arg1, const void *arg2, int type ) );
char *timestamp args( ( char strtime[16] ) );

/* db.c */
void	boot_db		args( ( void ) );
void	area_update	args( ( void ) );
CD *	create_mobile	args( ( MOB_INDEX_DATA *pMobIndex ) );
OD *	create_object	args( ( OBJ_INDEX_DATA *pObjIndex, int level ) );
void	clear_char	args( ( CHAR_DATA *ch ) );
void	free_char	args( ( CHAR_DATA *ch ) );
char *	get_extra_descr	args( ( const char *name, EXTRA_DESCR_DATA *ed ) );
MID *	get_mob_index	args( ( int vnum ) );
OID *	get_obj_index	args( ( int vnum ) );
RID *	get_room_index	args( ( int vnum ) );
void 	free_exit 			args ( ( EXIT_DATA *exit ) );
void 	free_extra_descr	args ( ( EXTRA_DESCR_DATA *ed ) );
void 	free_room			args ( ( ROOM_INDEX_DATA *room_from ) );

void	free_predelay	args( ( PREDELAY_DATA *p ) );
PREDELAY_DATA * new_predelay args( ( void ) );
int number_of_rooms args( ( AREA_DATA *pArea ) );
void obj_strings args( ( OBJ_DATA *obj ) );


//fileio
char	fread_letter	args( ( FILE *fp ) );
int	fread_number	args( ( FILE *fp ) );
char *	fread_string	args( ( FILE *fp ) );
char *fread_string_full args( ( FILE *fp, char terminator, bool fKillSpace ) );
void	fread_to_eol	args( ( FILE *fp ) );
char *	fread_word	args( ( FILE *fp ) );

//mem_manage
void *	alloc_mem	args( ( int sMem ) );
void *	alloc_perm	args( ( int sMem ) );
void	free_mem	args( ( void *pMem, int sMem ) );
char *	str_dup		args( ( const char *str ) );
void	free_string	args( ( char *pstr ) );
int	number_fuzzy	args( ( int number ) );
int	number_range	args( ( int from, int to ) );
int	number_percent	args( ( void ) );
int	number_door	args( ( void ) );
int	number_bits	args( ( int width ) );
long	number_mm	args( ( void ) );
int	dice		args( ( int number, int size ) );
int	interpolate	args( ( int level, int value_00, int value_32 ) );
void	smash_tilde	args( ( char *str ) );
bool	str_cmp		args( ( const char *astr, const char *bstr ) );
bool	str_prefix	args( ( const char *astr, const char *bstr ) );
bool	str_infix	args( ( const char *astr, const char *bstr ) );
bool	str_suffix	args( ( const char *astr, const char *bstr ) );
char *	capitalize	args( ( const char *str ) );
void	append_file	args( ( CHAR_DATA *ch, char *file, char *str ) );
void	bug		args( ( const char *str, int param ) );
void	log_string	args( ( const char *str ) );
void	tail_chain	args( ( void ) );

/* fight.c */
void	violence_update	args( ( void ) );
void	multi_hit	args( ( CHAR_DATA *ch, CHAR_DATA *victim, int dt ) );
void	damage		args( ( CHAR_DATA *ch, CHAR_DATA *victim, int dam,
			    int dt ) );
void	update_pos	args( ( CHAR_DATA *victim ) );
void	stop_fighting	args( ( CHAR_DATA *ch, bool fBoth ) );
void 	really_kill	args( ( CHAR_DATA *victim, bool dropStuff ) );
bool	is_safe		args( ( CHAR_DATA *ch, CHAR_DATA *victim ) );
void	one_hit		args( ( CHAR_DATA *ch, CHAR_DATA *victim, int dt, int ambush ) );


/* handler.c */
int get_hours_played  args( ( CHAR_DATA *ch ) );
int	get_trust	args( ( CHAR_DATA *ch ) );
int	get_age		args( ( CHAR_DATA *ch ) );
int	get_curr_Strength	args( ( CHAR_DATA *ch ) );
int	get_curr_Endurance	args( ( CHAR_DATA *ch ) );
int	get_curr_Dexterity	args( ( CHAR_DATA *ch ) );
int	get_curr_Speed	args( ( CHAR_DATA *ch ) );
int	get_curr_Willpower	args( ( CHAR_DATA *ch ) );
int get_curr_Potency args( ( CHAR_DATA *ch ) );
int	get_curr_Judgement	args( ( CHAR_DATA *ch ) );
int	get_curr_Intelligence	args( ( CHAR_DATA *ch ) );
int	get_curr_Wisdom	args( ( CHAR_DATA *ch ) );
int	get_curr_Charm	args( ( CHAR_DATA *ch ) );
int	body_weight	args( ( CHAR_DATA *ch ) );
int	can_carry_n	args( ( CHAR_DATA *ch ) );
int	can_carry_w	args( ( CHAR_DATA *ch ) );
bool	is_name		args( ( const char *str, char *namelist ) );
void	affect_to_char	args( ( CHAR_DATA *ch, AFFECT_DATA *paf ) );
void	affect_remove	args( ( CHAR_DATA *ch, AFFECT_DATA *paf ) );
void	affect_strip	args( ( CHAR_DATA *ch, int sn ) );
bool	is_affected	args( ( CHAR_DATA *ch, int sn ) );
void	affect_join	args( ( CHAR_DATA *ch, AFFECT_DATA *paf ) );
void	char_from_room	args( ( CHAR_DATA *ch ) );
void	char_to_room	args( ( CHAR_DATA *ch, ROOM_INDEX_DATA *pRoomIndex ) );
void	obj_to_char	args( ( OBJ_DATA *obj, CHAR_DATA *ch ) );
void    obj_to_free_hand args( ( OBJ_DATA *obj, CHAR_DATA *ch ) );
void	obj_from_char	args( ( OBJ_DATA *obj ) );
int	apply_ac	args( ( OBJ_DATA *obj, int iWear ) );
OD *	get_eq_char	args( ( CHAR_DATA *ch, int iWear ) );
void	equip_char	args( ( CHAR_DATA *ch, OBJ_DATA *obj, int iWear ) );
void	unequip_char	args( ( CHAR_DATA *ch, OBJ_DATA *obj ) );
int	count_obj_list	args( ( OBJ_INDEX_DATA *obj, OBJ_DATA *list ) );
void	obj_from_room	args( ( OBJ_DATA *obj ) );
void	obj_to_room	args( ( OBJ_DATA *obj, ROOM_INDEX_DATA *pRoomIndex ) );
void	obj_to_obj	args( ( OBJ_DATA *obj, OBJ_DATA *obj_to ) );
void	obj_from_obj	args( ( OBJ_DATA *obj ) );
void	extract_obj	args( ( OBJ_DATA *obj ) );
void	extract_char	args( ( CHAR_DATA *ch, bool fPull ) );
OD *	get_obj_in_carried_containers args( ( CHAR_DATA *ch, char *argument ) );
OD *	get_obj_in_room_containers args( ( CHAR_DATA *ch, char *argument ) );
OD *	get_obj_in_carried_containers2 args( ( CHAR_DATA *ch, char *argument ) );
OD *	get_obj_in_room_containers2 args( ( CHAR_DATA *ch, char *argument ) );
CD *	get_char_room	args( ( CHAR_DATA *ch, char *argument ) );
CD *	get_char_world	args( ( CHAR_DATA *ch, char *argument ) );
OD *	get_obj_type	args( ( OBJ_INDEX_DATA *pObjIndexData ) );
OD *	get_obj_list	args( ( CHAR_DATA *ch, char *argument,
			    OBJ_DATA *list ) );
OD *	get_obj_carry	args( ( CHAR_DATA *ch, char *argument ) );
OD *	get_obj_wear	args( ( CHAR_DATA *ch, char *argument ) );
OD *	get_obj_here	args( ( CHAR_DATA *ch, char *argument ) );
OD *	get_obj_close	args( ( CHAR_DATA *ch, char *argument ) );
OD *	get_obj_world	args( ( CHAR_DATA *ch, char *argument ) );
OD *	create_money	args( ( int amount ) );
int	get_obj_number	args( ( OBJ_DATA *obj ) );
int	get_obj_weight	args( ( OBJ_DATA *obj ) );
bool	room_is_dark	args( ( ROOM_INDEX_DATA *pRoomIndex ) );
bool	room_is_private	args( ( ROOM_INDEX_DATA *pRoomIndex ) );
bool	can_see		args( ( CHAR_DATA *ch, CHAR_DATA *victim ) );
bool	can_see_obj	args( ( CHAR_DATA *ch, OBJ_DATA *obj ) );
bool	can_drop_obj	args( ( CHAR_DATA *ch, OBJ_DATA *obj ) );
char *	item_type_name	args( ( OBJ_DATA *obj ) );
char *	affect_loc_name	args( ( int location ) );
char *	affect_bit_name	args( ( int vector ) );
char *	extra_bit_name	args( ( int extra_flags ) );
char *smash_article		args( ( char *text ) );
char *select_a_an		args( ( char *str ) );
void    printf_to_char  args( ( CHAR_DATA *ch, char *fmt, ...) ) __attribute__ ((format(printf, 2,3)));

void	set_predelay	args( ( CHAR_DATA *ch, int delay, DO_FUN *fnptr,
			 char *argument, int number, CHAR_DATA *victim1,
			 CHAR_DATA *victim2, OBJ_DATA *obj1, OBJ_DATA *obj2, bool noInterrupt ) );
			 

void obj_to_off_hand args( ( OBJ_DATA *obj, CHAR_DATA *ch ) );

/* interp.c */
void    show_roundtime    args(( CHAR_DATA *ch, int sec ) );
bool    add_roundtime    args(( CHAR_DATA *ch, int pulse ) );
void    show_stun    args(( CHAR_DATA *ch, int sec ) );
bool    add_stun   args(( CHAR_DATA *ch, int pulse ) );
void	interpret	args( ( CHAR_DATA *ch, char *argument ) );
bool	is_number	args( ( char *arg ) );
int	number_argument	args( ( char *argument, char *arg ) );
char *	one_argument	args( ( char *argument, char *arg_first ) );
char *  parse		args( ( char *argument, DO_FUN *cmd ) );



/* magic.c */
int	skill_lookup	args( ( const char *name ) );
int	spell_lookup	args( ( const char *name ) );
int	slot_lookup	args( ( int slot ) );
bool	saves_spell	args( ( int level, CHAR_DATA *victim ) );
void	obj_cast_spell	args( ( int sn, int level, CHAR_DATA *ch,
				    CHAR_DATA *victim, OBJ_DATA *obj ) );
bool 	can_cast args( ( CHAR_DATA *ch, short sn ) );

//olc_room.c

int get_direction args( ( char *arg ) );
bool can_build   args( ( CHAR_DATA *ch, AREA_DATA *pArea ) );

/* save.c */
void	save_char_obj	args( ( CHAR_DATA *ch ) );
bool	load_char_obj	args( ( DESCRIPTOR_DATA *d, char *name ) );

/* special.c */

/*
 * The following special functions are available for mobiles.
 */

DECLARE_SPEC_FUN(	spec_healer		);


/*
 * The following special functions are available for rooms.
 */
DECLARE_ROOM_SPECIAL(	spec_start_room		);
DECLARE_ROOM_SPECIAL(	spec_orchard		);
DECLARE_ROOM_SPECIAL(	spec_temple	);


/*
 * The following special functions are available for rooms.
 */
DECLARE_OBJ_FUN(  spec_armsheath	 );
DECLARE_OBJ_FUN(  spec_flame_blade	 );
DECLARE_OBJ_FUN(  spec_glow	 );
DECLARE_OBJ_FUN(  spec_trashbin	 );



SF *	spec_lookup	args( ( const char *name ) );
OBJ_FUN * obj_spec_lookup args( ( const char *name ) );
ROOM_SPECIAL * room_fun_lookup args( ( const char *name ) );

/* update.c */
int		get_exp_required args (( int level ));
int		get_total_exp_required args(( int level ));
void	advance_level	args( ( CHAR_DATA *ch ) );
void	gain_exp	args( ( CHAR_DATA *ch, int gain ) );
void	gain_condition	args( ( CHAR_DATA *ch, int iCond, int value ) );
void	update_handler	args( ( void ) );
int get_current_hour args( ( ) );
int get_current_minute args( ( ) );
int get_current_month args( ( ) );
int get_high_temp args( ( ) );
int get_low_temp args( ( ) );
int get_current_temp args( ( ) );

// gamesys


short  open_1d100 args ( ( void ) );
int 	random_hit_location args( ( void ) );
short 	get_divisor args( ( short ag ) );
char 	*ag_to_name args( ( short ag ) );
char 	*dt_to_name args( ( short dt ) );
short 	get_bolt_damage_factor args( ( short bolt_type, short armor_group ) );
short 	get_damage_factor args( ( short weapon_type, short armor_group ) );
short 	get_armor_hit_loc args( ( CHAR_DATA *ch, int hit_loc ) );
OBJ_DATA *get_armor_hit_obj args( ( CHAR_DATA *ch, int hit_loc ) );
int 	get_armor_flag args( ( int hit_loc ) );
int 	stat_bonus args( ( int stat, CHAR_DATA *ch, int  ) );
int 	skill_bonus args( ( int ranks ) );
short	warding_check args(( CHAR_DATA *ch, CHAR_DATA *victim ));
short	bolt_check args(( CHAR_DATA *ch, CHAR_DATA *victim, int sn ));
bool    man_check args(( CHAR_DATA *ch, CHAR_DATA *victim ));
void 	drop_hand args( ( CHAR_DATA *ch, short loc ) );
bool	aggro args( ( CHAR_DATA *ch ) );
bool	kill_aggro args( ( CHAR_DATA *ch ) );
bool	scavenge args( ( CHAR_DATA *ch ) );
bool	wander args( ( CHAR_DATA *ch ) );
bool 	spell_up args( ( CHAR_DATA *mob ) );
void 	init_maze args( ( ) );

void 	wound args( ( CHAR_DATA *ch, short loc, short severity, short bleed, bool silent ) );
OBJ_DATA *create_random_gem args( ( int level ) );
OBJ_DATA *create_random_weapon args( ( int level ) );
OBJ_DATA *create_random_armor args( ( int level ) );
OBJ_DATA *create_random_magic_item args( ( int num ) );
OBJ_DATA *create_random_box args( ( int level ) );
OBJ_DATA *create_random_container args( ( int level ) );
OBJ_DATA *create_random_shield args( ( int level ) );
void enchant_weapon args( ( OBJ_DATA *weapon, int enchant ) );
void enchant_armor args( ( OBJ_DATA *armor, int enchant ) );
void enchant_shield args( ( OBJ_DATA *shield, int enchant ) );

void mangle args(  ( CHAR_DATA *ch ) );
void rename_random args( ( CHAR_DATA *ch ) );

short get_max_field_exp args( ( CHAR_DATA *ch ) );

// track

void    hunt_victim     args( ( CHAR_DATA *ch ) );

// stealth

short armor_penalty args( ( CHAR_DATA *ch ) );


#undef	CD
#undef	MID
#undef	OD
#undef	OID
#undef	RID
#undef	SF


#if defined(macintosh)
#include <types.h>
#else
#include <sys/types.h>
#endif
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <time.h>
#include "merc.h"

/* Local functions */
bool	save_zone_file( AREA_DATA *pArea );

int get_direction( char *arg )
{
    if ( !str_cmp( arg, "n" ) || !str_cmp( arg, "north" ) ) return 0;
    if ( !str_cmp( arg, "e" ) || !str_cmp( arg, "east"  ) ) return 1;
    if ( !str_cmp( arg, "s" ) || !str_cmp( arg, "south" ) ) return 2;
    if ( !str_cmp( arg, "w" ) || !str_cmp( arg, "west"  ) ) return 3;
    if ( !str_cmp( arg, "u" ) || !str_cmp( arg, "up"    ) ) return 4;
    if ( !str_cmp( arg, "d" ) || !str_cmp( arg, "down"  ) ) return 5;
    if ( !str_cmp( arg, "ne" ) || !str_cmp( arg, "northeast" ) ) return 6;
    if ( !str_cmp( arg, "se" ) || !str_cmp( arg, "southeast"  ) ) return 7;
    if ( !str_cmp( arg, "nw" ) || !str_cmp( arg, "northwest" ) ) return 9;
    if ( !str_cmp( arg, "sw" ) || !str_cmp( arg, "southwest"  ) ) return 8;
    if ( !str_cmp( arg, "o" ) || !str_cmp( arg, "out"  ) ) return 10;

    return -1;
}




void write_string( FILE *fp, char *str )
{
    int ii;

    for (ii = 0; ; ii++)
    {
	switch ( str[ii] )
	{
	    case '\0':
		return;
		break;

	    case '\r':
		break;

	    case '~':
		fprintf( fp, "\\~" );
		break;

	    default:
		fprintf( fp, "%c", str[ii]);
		break;
	}
    }

    return;
}

bool can_build( CHAR_DATA *ch, AREA_DATA *pArea )
{
  if ( IS_NPC( ch ) )
    return FALSE;

  if ( pArea->builders == NULL )
    return FALSE;

  if( is_name( ch->name, "Bhuudo" ) ) // hehe :)
    return TRUE;

  if ( is_name( ch->name, pArea->builders ) )
    return TRUE;

  return FALSE;
}

void do_hset( CHAR_DATA *ch, char *argument )
{
    char keyword [MAX_INPUT_LENGTH];
    char arg2 [MAX_INPUT_LENGTH];
    HELP_DATA *pHelp;

    argument = one_argument( argument, keyword );
    argument = one_argument( argument, arg2 );

    if ( keyword[0] == '\0' || arg2[0] == '\0' )
    {
	send_to_char("Syntax:\n\r", ch );
	send_to_char("hset <keyword> <operation> (value/string)\n\r", ch );
	send_to_char("\noperation being one of the following:\n\r", ch );
	send_to_char("level, add, clear\n\r", ch );
	return;
    }

    for ( pHelp = help_first ; pHelp != NULL ; pHelp = pHelp->next )
    {
	if ( pHelp->level > get_trust( ch ) )
	    continue;

	if ( is_name( keyword, pHelp->keyword ) )
	    break;
    }

    if ( pHelp == NULL )
    {
	send_to_char("No such help file.\n\r", ch );
	return;
    }

    if ( !str_cmp( arg2, "level" ) )
    {
	int value;

	value = is_number( argument ) ? atoi( argument ) : -1;

	if (value > get_trust(ch) || value < 0)
	{
	    send_to_char("Level must be between 0 and your level.\n\r", ch );
	    return;
	}

	pHelp->level = value;

	send_to_char("Help file level set.\n\r", ch );
	return;
    }

    if ( !str_cmp( arg2, "add" ) )
    {
	char buf[MAX_STRING_LENGTH];

	strcpy( buf, pHelp->text );
	if ( strlen( buf ) + strlen( argument ) >= MAX_STRING_LENGTH-4 )
	{
	    send_to_char("Help too long to add to it.\n\r", ch );
	    return;
	}

	if ( buf[0] == '\0' )
	    strcpy( buf, argument );
	else
	    strcat( buf, argument );
	strcat( buf, "\n\r" );
	free_string( pHelp->text );
	pHelp->text = str_dup( buf );

	send_to_char("Text appended to help file.\n\r", ch );
	return;
    }

    if ( !str_cmp( arg2, "clear" ) )
    {
	char buf[MAX_STRING_LENGTH];

	buf[0] = '\0';

	free_string( pHelp->text );
	pHelp->text = str_dup( buf );

	send_to_char("Help file cleared.\n\r", ch );
	return;
    }

    do_hset( ch, "" );

    return;
}

void do_newhelp( CHAR_DATA *ch, char *argument )
{
    HELP_DATA *pHelp;
    extern int top_help;

    pHelp = (HELP_DATA *) alloc_perm( sizeof(*pHelp) );
    pHelp->level = get_trust( ch );
    pHelp->keyword = str_dup( argument );

    pHelp->text = str_dup( "\0" );

    pHelp->next = help_first;
    help_first = pHelp;
    top_help++;

    send_to_char("New help entry created.\n\r", ch );

    return;
}


void do_savehelps( CHAR_DATA *ch, char *argument )
{
    FILE *fp;
    HELP_DATA *pHelp;

    fclose( fpReserve );
    if ( ( fp = fopen( "test.helps", "w" )) == NULL )
    {
	send_to_char("Error opening help file.\n\r", ch );
	fpReserve = fopen( NULL_FILE, "r" );
	return;
    }

    fprintf( fp, "#HELPS\n\n\n\n" );

    for ( pHelp = help_first ; pHelp != NULL ; pHelp = pHelp->next )
    {
	if ( pHelp->keyword == NULL )
	    continue;

	if ( pHelp->keyword[0] == '\0' )
	    continue;

	if ( pHelp->text == NULL )
	    continue;

	if ( pHelp->text[0] == '\0' )
	    continue;

	fprintf( fp, "%d ",
	    pHelp->level );

	write_string( fp, pHelp->keyword );
	fprintf( fp, "~\n" );
	write_string( fp, pHelp->text );

	fprintf( fp, "~\n\n\n" );
    }

    fprintf( fp, "0 $~\n\n#$\n" );

    fclose( fp );
    fpReserve = fopen( NULL_FILE, "r" );
    system( "cp test.helps help.are" );
    system( "rm test.helps" );

    send_to_char("Help file written.\n\r", ch );

    return;
}


//do not save if there are objs/mobs/specials/resets, as those are not written...
//adding some safeguards
void do_savezone( CHAR_DATA *ch, char *argument )
{
    //char buf[MAX_STRING_LENGTH];
    AREA_DATA *pArea;
    //extern AREA_DATA *area_first;

    if ( IS_NPC( ch ) )
	return;

/*
    if ( get_trust( ch ) == MAX_LEVEL && !str_cmp( argument, "world" ) )
    {
	send_to_char("Saving all zones.\n\r", ch );

	for ( pArea = area_first; pArea != NULL; pArea = pArea->next )
	{
	    if ( save_zone_file( pArea ) == TRUE )
	    {
		sprintf( buf, "%s.are saved.\n\r", pArea->filename );
		send_to_char( buf, ch );
	    }
	    else
	    {
		sprintf( buf, "%s.are save FAILED.\n\r", pArea->filename );
		log_string( buf );
		send_to_char( buf, ch );
	    }
	}

	send_to_char("World save completed.\n\r", ch );
	return;
    }
*/

    if ( !can_build( ch, ch->in_room->area ) )
    {
	send_to_char(
	  "You do not have authorization to build in this zone.\n\r", ch );
	return;
    }
	
	if( str_cmp( argument, "confirm" ) )
	{
		printf_to_char(ch, "You must SAVEZONE CONFIRM to save ONLY THE ROOMS of this area as %s.are\r\n", ch->in_room->area->filename);
		
		return;
	}

    if (ch->in_room == NULL || (pArea = ch->in_room->area) == NULL)
    {
	send_to_char("Can't save zone, null area.\n\r", ch );
	return;
    }

    if ( save_zone_file( pArea ) == TRUE )
    {
	send_to_char("Zone file written.\n\r", ch );
    }
    else
    {
	send_to_char("Error writing zone file.\n\r", ch );
    }

    return;
}


bool save_zone_file( AREA_DATA *pArea )
{
	ROOM_INDEX_DATA *pRoomIndex;
    FILE *fp;
    EXIT_DATA *pExit;
    EXTRA_DESCR_DATA *ed;
    char fname[MAX_STRING_LENGTH];
    char buf[MAX_STRING_LENGTH];
    int iDoor;
	int i;

    if (pArea->filename == NULL || pArea->filename[0] == '\0')
    {
/*
	send_to_char("Can't save zone, null file name.\n\r", ch );
*/
	return -1;
    }

    sprintf(fname, "%s.tmp", pArea->filename);

    fclose( fpReserve );
    if ( ( fp = fopen( fname, "w" )) == NULL )
    {
/*
	send_to_char("Error opening output file.\n\r", ch );
*/
	fpReserve = fopen( NULL_FILE, "r" );
	return -1;
    }

    fprintf( fp, "#AREA " );
    write_string( fp, pArea->name );
    fprintf( fp, "~\n" );
	write_string( fp, pArea->builders );
    fprintf( fp, "~\n" );
    write_string( fp, pArea->filename );
    fprintf( fp, "~\n" );
    fprintf( fp, "%d %d\n", pArea->vnum_start, pArea->vnum_final );
	for( i = 0; i < 10; i++ )
	{
	write_string( fp, pArea->desc[i] );
	fprintf( fp, "~\n" );
	}
	
	fprintf( fp, "%d ", pArea->creatures[0] ? pArea->creatures[0] : -1 );
	fprintf( fp, "%d ", pArea->creatures[1] ? pArea->creatures[1] : -1 );
	fprintf( fp, "%d ", pArea->creatures[2] ? pArea->creatures[2] : -1 );
	fprintf( fp, "%d ", pArea->creatures[3] ? pArea->creatures[3] : -1 );
	fprintf( fp, "%d ", pArea->creatures[4] ? pArea->creatures[4] : -1 );
	
    //pArea->age		= 15; // other 'fields' not listed in *.ARE files
    //pArea->nplayer	= 0;
	//pArea->ncritter	= 0;
	
	fprintf( fp, "\n\n" );

    fprintf( fp, "#ROOMS\n" );
	for ( pRoomIndex = pArea->room_first; pRoomIndex != NULL; pRoomIndex = pRoomIndex->zone_next )
	{
	    if ( pRoomIndex->area != pArea )
		continue;

	    if ( pRoomIndex->vnum == 0)
		continue;

	    fprintf( fp, "#%d\n", pRoomIndex->vnum );
	    write_string( fp, pRoomIndex->name );
	    fprintf( fp, "~\n" );
	    write_string( fp, pRoomIndex->description );
	    fprintf( fp, "~\n" );
	    fprintf( fp, "0 %d %d\n",
		pRoomIndex->room_flags, pRoomIndex->sector_type );

	    for ( iDoor = 0; iDoor <= 10; iDoor++ )
	    {
		if ( ( pExit = pRoomIndex->exit[iDoor] ) == NULL )
		    continue;

		fprintf( fp, "D%d\n", iDoor );
		write_string( fp, pExit->description );
		fprintf( fp, "~\n" );
		write_string( fp, pExit->keyword );
		fprintf( fp, "~\n" );
		fprintf( fp, "0 0 %d\n", pExit->vnum );
	    }

	    for ( ed = pRoomIndex->extra_descr; ed != NULL; ed = ed->next )
	    {
		if ( ed->keyword == NULL || ed->description == NULL )
		    continue;

		fprintf( fp, "E\n" );
		write_string( fp, ed->keyword );
		fprintf( fp, "~\n" );
		write_string( fp, ed->description );
		fprintf( fp, "~\n" );
	    }

	    fprintf( fp, "S\n" );
	}

    fprintf( fp, "#0\n\n\n" );

    fprintf( fp, "#$\n" );

    fclose( fp );
    fpReserve = fopen( NULL_FILE, "r" );
    sprintf( buf, "cp %s.tmp %s.are", pArea->filename,
	pArea->filename );
    system( buf );
    sprintf( buf, "rm %s.tmp", pArea->filename );
    system( buf );

    return 1;
}





void do_break( CHAR_DATA *ch, char *argument )
{
    ROOM_INDEX_DATA *room = ch->in_room;
    int direction = -1;

    if ( IS_NPC( ch ) )
	return;

    if ( !can_build( ch, ch->in_room->area ) )
    {
	send_to_char(
	  "You don't have authorization to build in this zone.\n\r", ch );
	return;
    }

    if ( !str_cmp( argument, "all" ) )
    {
	for ( direction = 0; direction <= 10; direction++ )
	{
	    if ( room->exit[direction] == NULL )
		continue;

	    if ( room->exit[direction]->keyword[0] == '\0'
		&& room->exit[direction]->description[0] == '\0' )
	    {
		free_exit( room->exit[direction] );
		room->exit[direction] = NULL;
	    }
	    else
	    {
		room->exit[direction]->to_room = NULL;
		room->exit[direction]->vnum = 0;
		room->exit[direction]->key = 0;
	    }
	}

	send_to_char("All exits broken.\n\r", ch );
	return;
    }

    direction = get_direction( argument );

    if ( direction == -1 )
    {
	send_to_char("Syntax:   break <direction>  (north/east, etc.  or all).\n\r", ch );
	return;
    }

    if ( room->exit[direction] == NULL )
    {
	send_to_char("No exit in that direction.\n\r", ch );
	return;
    }

    if ( room->exit[direction]->keyword[0] == '\0'
    && room->exit[direction]->description[0] == '\0' )
    {
	free_exit( room->exit[direction] );
	room->exit[direction] = NULL;
    }
    else
    {
	room->exit[direction]->to_room = NULL;
	room->exit[direction]->vnum = 0;
	room->exit[direction]->key = 0;
    }

    send_to_char("Exit removed.\n\r", ch );
    return;
}


void do_collaps( CHAR_DATA *ch, char *argument )
{
    send_to_char("You need to type collapse all the way out.\n\r", ch );
    return;
}

void do_collapse( CHAR_DATA *ch, char *argument )
{
    if ( IS_NPC( ch ) )
	return;

    if ( !can_build( ch, ch->in_room->area ) )
    {
	send_to_char("You don't have authorization to build in this zone.\n\r", ch );
	return;
    }

    if ( ch->in_room->vnum == ROOM_VNUM_CHAT )
    {
	send_to_char("You can't collapse the chat room.\n\r", ch );
	return;
    }

    act("$n collapses the room.", ch, NULL, NULL, TO_ROOM );

    free_room( ch->in_room );

    send_to_char("Room deleted.\n\r", ch );
    return;
}

void do_connect( CHAR_DATA *ch, char *argument )
{
    char arg1 [MAX_INPUT_LENGTH];
    char arg2 [MAX_INPUT_LENGTH];
    extern short rev_dir[];
    int vnum;
    int direction = -1;
    bool fOneway = FALSE;
    ROOM_INDEX_DATA *pRoomIndex;
    EXIT_DATA *pexit = NULL;

    if ( IS_NPC( ch ) )
	return;

    if ( !can_build( ch, ch->in_room->area ) )
    {
	send_to_char(
	  "You do not have authorization to build in this zone.\n\r", ch );
	return;
    }

    argument = one_argument( argument, arg1 );
    argument = one_argument( argument, arg2 );

    if ( arg1[0] == '\0' || arg2[0] == '\0' )
    {
	send_to_char(" Syntax:  connect <direction> <vnum> [oneway]\n\r", ch );
	send_to_char(" [oneway] is optional parameter.  Exits are two way\n\r", ch );
	send_to_char(" by default.  Anything other than 'oneway' in that\n\r", ch );
	send_to_char(" field will be ignored.\n\r", ch );
	return;
    }

    if ( !is_number( arg2 ) )
    {
	send_to_char("Target room vnum must be numeric.\n\r", ch );
	return;
    }
    vnum = atoi( arg2 );

    direction = get_direction( arg1 );

    if ( direction == -1 )
    {
	do_connect( ch, "" );
	return;
    }

    if ( !str_cmp( argument, "oneway" ) || !str_cmp( argument, "one" ) )
	fOneway = TRUE;

    if ( (pRoomIndex = get_room_index( vnum )) == NULL )
    {
	send_to_char("Target room number does not exist.\n\r", ch );
	return;
    }

    if ( (pexit = ch->in_room->exit[direction]) == NULL )
    {
	pexit = (EXIT_DATA *) alloc_perm( sizeof(*pexit) );
	pexit->to_room = pRoomIndex;
	pexit->description = str_dup( "\0" );
	pexit->keyword = str_dup( "\0" );
	pexit->key = 0;
	pexit->vnum = vnum;
	ch->in_room->exit[direction] = pexit;
    }
    else
    {
	if ( pexit->to_room != NULL )
	{
	    send_to_char("Exit already exists.\n\r", ch );
	    return;
	}

	pexit->to_room = pRoomIndex;
	pexit->vnum = vnum;
    }

    if ( !fOneway )
    {
	if ( (pexit = pRoomIndex->exit[rev_dir[direction]]) == NULL )
	{
	    pexit = (EXIT_DATA *) alloc_perm( sizeof(*pexit) );
	    pexit->to_room = ch->in_room;
	    pexit->description = str_dup( "\0" );
	    pexit->keyword = str_dup( "\0" );
	    pexit->key = 0;
	    pexit->vnum = ch->in_room->vnum;
	    pRoomIndex->exit[rev_dir[direction]] = pexit;
	}
	else
	{
	    if ( pexit->to_room != NULL )
	    {
		send_to_char("Reverse exit already exists.  Forward entrance only created.\n\r", ch );
		return;
	    }

	    pexit->to_room = ch->in_room;
	    pexit->vnum = ch->in_room->vnum;
	}
    }

    send_to_char("Exit connected.\n\r", ch );
    char_from_room( ch );
    char_to_room( ch, pRoomIndex );
    do_look( ch, "auto" );
    return;
}


ROOM_INDEX_DATA *new_room( AREA_DATA *pArea, int vnum, char *room_name )
{
  ROOM_INDEX_DATA *pRoomIndex;
  extern ROOM_INDEX_DATA *room_free;
  extern ROOM_INDEX_DATA *room_index_hash[MAX_KEY_HASH];
  extern int top_room;
  int iHash;

  if ( room_free == NULL ) {
    pRoomIndex = (ROOM_INDEX_DATA *) alloc_perm( sizeof(*pRoomIndex));
  } else {
    pRoomIndex = room_free;
    room_free = pRoomIndex->next;
  }

  if ( pRoomIndex == NULL ) {
    log_string( "Error opening new room." );
    return NULL;
  }

  pRoomIndex->people = NULL;
  pRoomIndex->contents = NULL;
  pRoomIndex->extra_descr = NULL;
  pRoomIndex->area = pArea;
  pRoomIndex->vnum = vnum;

  if ( room_name == NULL || room_name[0] == '\0' )
    pRoomIndex->name = str_dup( "" );
  else
    pRoomIndex->name = str_dup( room_name );

  pRoomIndex->description = str_dup( "" );
  pRoomIndex->room_flags = 0;
  pRoomIndex->sector_type = SECT_CITY;

  iHash = vnum % MAX_KEY_HASH;
  pRoomIndex->next = room_index_hash[iHash];
  room_index_hash[iHash] = pRoomIndex;
  pRoomIndex->zone_next = NULL;
  top_room++;

  if ( pArea->room_first == NULL )
  {
    pRoomIndex->area->room_first = pRoomIndex;
    pRoomIndex->area->room_last = pRoomIndex;
  }
  else
  {
    pRoomIndex->area->room_last->zone_next = pRoomIndex;
    pRoomIndex->area->room_last = pRoomIndex;
  }

  return pRoomIndex;
}


void do_dig( CHAR_DATA *ch, char *argument )
{
  extern short rev_dir[];
  int door;
  int direction;
  int vnum;
  bool fEonly = FALSE;
  char arg1 [MAX_INPUT_LENGTH];
  char arg2 [MAX_INPUT_LENGTH];
  char buf [MAX_STRING_LENGTH];
  ROOM_INDEX_DATA *pRoomIndex;
  EXIT_DATA *pexit = NULL;

  if ( IS_NPC( ch ) )
    return;

  if ( !can_build( ch, ch->in_room->area ) ) {
    send_to_char(
      "You do not have authorization to build in this zone.\n\r", ch );
    return;
  }

  argument = one_argument( argument, arg1 );

  if ( arg1[0] != '\0' && !str_cmp( arg1, "list" ) ) {
    int col = 0;

    sprintf( buf, "Room vnums in use for %s.\n\r", ch->in_room->area->name );
    send_to_char( buf, ch );

    for ( pRoomIndex = ch->in_room->area->room_first;
      pRoomIndex != NULL; pRoomIndex = pRoomIndex->zone_next ) {

      sprintf( buf, " %6d", pRoomIndex->vnum );
      send_to_char( buf, ch );
      if ( ( ++col % 10 ) == 0 )
        send_to_char( "\n\r", ch );
    }

    if ( col % 10 != 0 )
      send_to_char( "\n\r", ch );

    return;
  }

  argument = one_argument( argument, arg2 );

  if ( arg1[0] == '\0' || arg2[0] == '\0' ) {
    send_to_char(" Syntax:  dig new <vnum> [room name]\n\r", ch );
    send_to_char("          dig (north/south/east etc) <vnum> [room name]\n\r",
      ch );
    send_to_char("          dig list\n\r", ch );
    return;
  }

  if ( !is_number( arg2 ) ) {
    send_to_char("Vnum must be numeric.\n\r", ch );
    return;
  }
  vnum = atoi( arg2 );

/*  add vnum checking    */
  if ( get_room_index( vnum ) != NULL ) {
    send_to_char("Vnums can not be duplicated.\n\r", ch );
    return;
  }

  if ( vnum < ch->in_room->area->vnum_start
    || vnum > ch->in_room->area->vnum_final ) {
    send_to_char("Vnum out of range.\n\r", ch );
    sprintf( buf, "Vnum must be in range %d to %d.\n\r",
      ch->in_room->area->vnum_start,
      ch->in_room->area->vnum_final );
    send_to_char( buf, ch );
    return;
  }

  direction = 20;

  if ( !str_cmp( arg1, "new" ) )
    direction = -1;

  direction = get_direction( arg1 );

  if ( direction > 10 ) {
    do_dig( ch, "" );
    return;
  }

  if ( direction != -1 && ((pexit = ch->in_room->exit[direction]) != NULL)) {
    if ( pexit->to_room != NULL ) {
      send_to_char("There is already an exit in that direction.\n\r", ch );
      return;
    }
    fEonly = TRUE;
  }

  if ( ( pRoomIndex = new_room( ch->in_room->area, vnum, argument ) ) == NULL )
    return;

  for ( door = 0; door <= 10; door++ ) {
    pRoomIndex->exit[door] = NULL;
  }

  if ( direction != -1 ) {
    if (fEonly) {
      pexit->to_room = pRoomIndex;
    } else {
/****fixmefixme****/
      pexit = (EXIT_DATA *) alloc_mem( sizeof(*pexit) );
      pexit->to_room = pRoomIndex;
      pexit->description = str_dup( "\0" );
      pexit->keyword = str_dup( "\0" );
      pexit->key = 0;
      pexit->vnum = vnum;
      ch->in_room->exit[direction] = pexit;
    }

    pexit = (EXIT_DATA *) alloc_perm( sizeof(*pexit) );
    pexit->to_room = ch->in_room;
    pexit->description = str_dup( "\0" );
    pexit->keyword = str_dup( "\0" );
    pexit->key = 0;
    pexit->vnum = ch->in_room->vnum;
    pRoomIndex->exit[rev_dir[direction]] = pexit;
  }

  send_to_char("Room created.\n\r", ch );
  char_from_room( ch );
  char_to_room( ch, pRoomIndex );
  do_look( ch, "auto" );
  return;
}


void do_exset( CHAR_DATA *ch, char *argument )
{
    char direction [MAX_INPUT_LENGTH];
    char command [MAX_INPUT_LENGTH];
    char buf [MAX_STRING_LENGTH];
    EXIT_DATA *pexit;
    extern short rev_dir[];
    int door = -1;

    if ( IS_NPC( ch ) )
	return;

    if ( !can_build( ch, ch->in_room->area ) )
    {
	send_to_char("You don't have authorization to build in this zone.\n\r", ch );
	return;
    }

    argument = one_argument( argument, direction );
    argument = one_argument( argument, command );

    if ( direction[0] == '\0' || command[0] == '\0' )
    {
	send_to_char("Syntax:\n\r", ch );
	send_to_char(" exset <direction> <command> <argument>\n\r", ch );
	send_to_char("\n\r", ch );
	send_to_char("Where argument is one of the following:\n\r", ch );
	send_to_char("desc cldesc key door/flags keyword lock/diff\n\r", ch );
	return;
    }

    door = get_direction( direction );

    if ( door == -1 )
    {
	send_to_char("Invalid direction.\n\r", ch );
	return;
    }

    if ( (pexit = ch->in_room->exit[door]) == NULL )
    {
	send_to_char("No exit data in that direction.  Creating entry.\n\r", ch );

	pexit = (EXIT_DATA *) alloc_perm( sizeof( *pexit ) );
	pexit->to_room = NULL;
	pexit->description = str_dup( "\0" );
	pexit->keyword = str_dup( "\0" );
	pexit->key = 0;
	pexit->vnum = -1;
	ch->in_room->exit[door] = pexit;
    }

    if ( !str_cmp( command, "cldesc" ) )
    {
	buf[0] = '\0';

	free_string( pexit->description );
	pexit->description = str_dup( buf );

	send_to_char("Exit description cleared.\n\r", ch );
	return;
    }

    if ( !str_cmp( command, "desc" ) )
    {
	strcpy( buf, pexit->description );

	if ( strlen( buf ) + strlen( argument ) >= MAX_STRING_LENGTH -4 )
	{
	    send_to_char("Exit description too long, text not added.\n\r", ch );
	    return;
	}

	if ( buf[0] == '\0' )
	    strcpy( buf, argument );
	else
	    strcat( buf, argument );
	strcat( buf, "\n\r" );
	free_string( pexit->description );
	pexit->description = str_dup( buf );

	send_to_char("Text added to exit description.\n\r", ch );
	return;
    }

    if ( !str_cmp( command, "keyword" ) )
    {
	if ( argument == NULL )
	{
	    send_to_char("Set what as keyword?\n\r", ch );
	    return;
	}
	free_string( pexit->keyword );
	pexit->keyword = str_dup( argument );
	return;
    }

    do_exset( ch, "" );

    return;
}


void do_rdesc( CHAR_DATA *ch, char *argument )
{
    char buf[MAX_STRING_LENGTH];
    char arg1 [MAX_INPUT_LENGTH];

    if ( IS_NPC( ch ) )
	return;

    if ( !can_build( ch, ch->in_room->area ) )
    {
	send_to_char(
	  "You do not have authorization to build in this zone.\n\r", ch );
	return;
    }

    argument = one_argument( argument, arg1 );

    if ( arg1[0] == '\0' )
    {
	send_to_char(" Syntax:  rdesc clear\n\r",		ch );
	send_to_char("          rdesc add <text to add>\n\r",	ch );
	return;
    }

    if ( !str_cmp( arg1, "add" ) )
    {
	strcpy( buf, ch->in_room->description );

	if ( strlen( buf ) + strlen( argument ) >= MAX_STRING_LENGTH -4 )
	{
	    send_to_char("Room description too long, text not added.\n\r", ch );
	    return;
	}

	if ( buf[0] == '\0' )
	    strcpy( buf, argument );
	else
	    strcat( buf, argument );
	strcat( buf, "\n\r" );
	free_string( ch->in_room->description );
	ch->in_room->description = str_dup( buf );

	send_to_char("Text added to room description.\n\r", ch );
	return;
    }

    if ( !str_cmp( arg1, "clear" ) )
    {
	buf[0] = '\0';

	free_string( ch->in_room->description );
	ch->in_room->description = str_dup( buf );

	send_to_char("Room description cleared.\n\r", ch );
	return;
    }

    do_rdesc( ch, "" );

    return;
}


void do_rextra( CHAR_DATA *ch, char *argument  )
{
    char keyword[MAX_INPUT_LENGTH];
    char command[MAX_INPUT_LENGTH];
    char buf[MAX_STRING_LENGTH];
    EXTRA_DESCR_DATA *ed;

    if ( IS_NPC( ch ) )
	return;

    if ( !can_build( ch, ch->in_room->area ) )
    {
	send_to_char(
	  "You do not have authorization to build in this zone.\n\r", ch );
	return;
    }

    argument = one_argument( argument, keyword );
    argument = one_argument( argument, command );

    if ( keyword[0] == '\0' || command[0] == '\0' )
    {
	send_to_char("Syntax:  rextra <keyword> <command> [text]\n\r", ch );
	send_to_char("\n\r", ch );
	send_to_char("Where command is one of:\n\r", ch );
	send_to_char(" clear create desc remove\n\r", ch );
	return;
    }

    for ( ed = ch->in_room->extra_descr; ed != NULL; ed = ed->next )
    {
	if ( is_name( keyword, ed->keyword ) )
	    break;
    }

    if ( !str_cmp( command, "create" ) )
    {
	if ( ed != NULL )
	{
	    send_to_char("Extra description already exists.\n\r", ch );
	    return;
	}

	if ( argument[0] == '\0' )
	{
    send_to_char("What should the extra description be called?\n\r", ch );
	    return;
	}

	ed = (EXTRA_DESCR_DATA *) alloc_perm( sizeof( *ed ) );

	ed->keyword = str_dup( argument );
	ed->description = str_dup( "\0" );

	ed->next = ch->in_room->extra_descr;
	ch->in_room->extra_descr = ed;

	send_to_char("Extra description created.\n\r", ch );
	return;
    }

    if ( ed == NULL )
    {
	send_to_char("No such extra description.\n\r", ch );
	return;
    }

    if ( !str_cmp( command, "clear" ) )
    {
	free_string( ed->description );
	ed->description = str_dup( "\0" );

	send_to_char("Extra description cleared.\n\r", ch );
	return;
    }

    if ( !str_cmp( command, "desc" ) )
    {
	strcpy( buf, ed->description );

	if ( strlen( buf ) + strlen( argument ) >= MAX_STRING_LENGTH -4 )
	{
	    send_to_char("Extra description too long, text not added.\n\r", ch );
	    return;
	}

	if ( buf[0] == '\0' )
	    strcpy( buf, argument );
	else
	    strcat( buf, argument );
	strcat( buf, "\n\r" );
	free_string( ed->description );
	ed->description = str_dup( buf );

	send_to_char("Text added to extra description.\n\r", ch );
	return;
    }

    if ( !str_cmp( command, "remove" ) )
    {
	EXTRA_DESCR_DATA *pextra;

	if ( ch->in_room->extra_descr == ed )
	{
	    ch->in_room->extra_descr = ed->next;
	}
	else
	{
	    for ( pextra = ch->in_room->extra_descr; pextra != NULL; pextra = pextra->next )
	    {
		if ( pextra->next == ed )
		    break;
	    }

	    pextra->next = ed->next;
	}

	free_extra_descr( ed );

	send_to_char("Extra description removed.\n\r", ch );
	return;
    }

    do_rextra( ch, "" );
    return;
}

void do_ddesc( CHAR_DATA *ch, char *argument )
{
	int i;
	char arg1 [MAX_INPUT_LENGTH];
    char arg2 [MAX_INPUT_LENGTH];
	int value;

    smash_tilde( argument );
    argument = one_argument( argument, arg1 );
    strcpy( arg2, argument );
	
	if ( arg1[0] == '\0' )
	{
	for( i = 0; i < 10; i++ )
	{
	printf_to_char(ch, "%2d: %s\r\n", i, ch->in_room->area->desc[i] );
	}
	return;
	}
	
    if ( !is_number( arg1 ) || arg2[0] == '\0' )
    {
	send_to_char( "Syntax: ddesc [number] [description].\r\n", ch );
	return;
    }
    value = atoi( arg1 );
	ch->in_room->area->desc[value]	= str_dup( arg2 );
	
	printf_to_char( ch, "Dynamic description %d set to: %s\r\n", value, ch->in_room->area->desc[value] );
}


#if defined(macintosh)
#include <types.h>
#else
#include <sys/types.h>
#endif
#include <ctype.h>
#include <stdio.h>
#include <string.h>
#include <time.h>
#include "merc.h"

#if !defined(macintosh)
extern	int	_filbuf		args( (FILE *) );
#endif



/*
 * Array of containers read for proper re-nesting of objects.
 */
#define MAX_NEST	100
static	OBJ_DATA *	rgObjNest	[MAX_NEST];



/*
 * Local functions.
 */
void	fwrite_char	args( ( CHAR_DATA *ch,  FILE *fp ) );
void	fwrite_obj	args( ( CHAR_DATA *ch,  OBJ_DATA  *obj,
			    FILE *fp, int iNest ) );
void	fread_char	args( ( CHAR_DATA *ch,  FILE *fp ) );
void	fread_obj	args( ( CHAR_DATA *ch,  FILE *fp ) );



/*
 * Save a character and inventory.
 * Would be cool to save NPC's too for quest purposes,
 *   some of the infrastructure is provided.
 */
void save_char_obj( CHAR_DATA *ch )
{
    char strsave[MAX_INPUT_LENGTH];
    FILE *fp;

    if ( IS_NPC(ch) || !IS_SET(ch->act, PLR_CREATED ) )
	return;
	
    if ( ch->desc != NULL && ch->desc->original != NULL )
	ch = ch->desc->original;

    ch->save_time = current_time;
    fclose( fpReserve );
    sprintf( strsave, "%s%s", PLAYER_DIR, capitalize( ch->name ) );
    if ( ( fp = fopen( strsave, "w" ) ) == NULL )
    {
	bug( "Save_char_obj: fopen", 0 );
	perror( strsave );
    }
    else
    {
	fwrite_char( ch, fp );
	
	if ( ch->carrying != NULL )
	    fwrite_obj( ch, ch->carrying, fp, 0 );
	
	fprintf( fp, "#END\n" );
    }
    fclose( fp );
	
		    fpReserve = fopen( NULL_FILE, "r" );
			
    return;
}



/*
 * Write the char.
 */
void fwrite_char( CHAR_DATA *ch, FILE *fp )
{
    AFFECT_DATA *paf;
    int sn;
	
    fprintf( fp, "#%s\n", IS_NPC(ch) ? "MOB" : "PLAYER"		);

    fprintf( fp, "Name         %s~\n",	ch->name		);
    fprintf( fp, "ShortDescr   %s~\n",	ch->short_descr		);
    fprintf( fp, "LongDescr    %s~\n",	ch->long_descr		);
    fprintf( fp, "Description  %s~\n",	ch->description		);
	
	if( !IS_NPC(ch ) )
	{
			    fprintf( fp, "Age  %d\n",	ch->pcdata->age	);
	    fprintf( fp, "Hair  %s~\n",	ch->pcdata->hair	);
		    fprintf( fp, "Eyes  %s~\n",	ch->pcdata->eyes		);
			    fprintf( fp, "Skin  %s~\n",	ch->pcdata->skin	);
	}
    fprintf( fp, "Sex          %d\n",	ch->sex			);
	fprintf( fp, "Stance       %d\n",	ch->stance			);
    fprintf( fp, "Class        %d\n",	ch->class		);
    fprintf( fp, "Race         %d\n",	ch->race		);
    fprintf( fp, "Level        %d\n",	ch->level		);
    fprintf( fp, "Trust        %d\n",	ch->trust		);
    fprintf( fp, "Played       %d\n",
	ch->played + (int) (current_time - ch->logon)		);
    fprintf( fp, "Room         %d\n",
	(  ch->in_room == get_room_index( ROOM_VNUM_LIMBO )
	&& ch->was_in_room != NULL )
	    ? ch->was_in_room->vnum
	    : ch->in_room->vnum );

    fprintf( fp, "HpManaMove   %d %d %d %d %d %d\n",
	ch->hit, ch->max_hit, ch->mana, ch->max_mana, ch->move, ch->max_move );
    fprintf( fp, "Gold         %d\n",	ch->gold		);
	
	if(!IS_NPC(ch) )
	fprintf( fp, "Bank         %d\n",	ch->pcdata->bank		);
    
	fprintf( fp, "Exp          %d\n",	ch->exp			);
	
	if(!IS_NPC(ch) )
	fprintf( fp, "FieldExp     %d\n",	ch->pcdata->fieldExp			);

    fprintf( fp, "Act          %d\n",   ch->act			);
    fprintf( fp, "AffectedBy   %d\n",	ch->affected_by		);
    /* Bug fix from Alander */
    fprintf( fp, "Position     %d\n",   ch->position );

    fprintf( fp, "Practice     %d\n",	ch->practice		);
    fprintf( fp, "SavingThrow  %d\n",	ch->saving_throw	);
    fprintf( fp, "Hitroll      %d\n",	ch->hitroll		);
    fprintf( fp, "Damroll      %d\n",	ch->damroll		);
    fprintf( fp, "Armor        %d\n",	ch->armor		);
    fprintf( fp, "Deaf         %d\n",	ch->deaf		);
	fprintf( fp, "Deeds        %d\n",	ch->pcdata->deeds		);
	fprintf( fp, "Decay        %d\n",	ch->decay		);
	fprintf( fp, "Wrap         %d\n",	ch->pcdata->wrap		);

    if ( IS_NPC(ch) )
    {
	fprintf( fp, "Vnum         %d\n",	ch->pIndexData->vnum	);
    }
    else
    {
	fprintf( fp, "Password     %s~\n",	ch->pcdata->pwd		);
	fprintf( fp, "Bamfin       %s~\n",	ch->pcdata->bamfin	);
	fprintf( fp, "Bamfout      %s~\n",	ch->pcdata->bamfout	);
	fprintf( fp, "Title        %s~\n",	ch->pcdata->title	);
	fprintf( fp, "AttrPerm     %d %d %d %d %d %d %d %d %d %d\n",
	    ch->pcdata->perm_Strength,
	    ch->pcdata->perm_Endurance,
	    ch->pcdata->perm_Dexterity,
	    ch->pcdata->perm_Speed,
	    ch->pcdata->perm_Willpower,
		ch->pcdata->perm_Potency,
	    ch->pcdata->perm_Judgement,
	    ch->pcdata->perm_Intelligence,
	    ch->pcdata->perm_Wisdom,
	    ch->pcdata->perm_Charm
		);

	fprintf( fp, "AttrMod      %d %d %d %d %d %d %d %d %d %d\n",
	    ch->pcdata->mod_Strength,
	    ch->pcdata->mod_Endurance,
	    ch->pcdata->mod_Dexterity,
	    ch->pcdata->mod_Speed,
	    ch->pcdata->mod_Willpower,
		ch->pcdata->mod_Potency,
	    ch->pcdata->mod_Judgement,
	    ch->pcdata->mod_Intelligence,
	    ch->pcdata->mod_Wisdom,
	    ch->pcdata->mod_Charm
		);
		
	fprintf( fp, "BaseStats		%d %d %d %d\n",
	    ch->as,
		ch->ds,
		ch->cs,
		ch->td
		);
		
		
	fprintf( fp, "Body      %d %d %d %d %d %d %d %d %d %d %d %d %d %d %d %d %d\n",
	    ch->body[BODY_NONE],
		ch->body[BODY_R_EYE],
		ch->body[BODY_L_EYE],
		ch->body[BODY_HEAD],
		ch->body[BODY_NECK],
		ch->body[BODY_CHEST],
		ch->body[BODY_BACK],
		ch->body[BODY_R_ARM],
		ch->body[BODY_R_HAND],
		ch->body[BODY_L_ARM],
		ch->body[BODY_L_HAND],
		ch->body[BODY_ABDOMEN],
		ch->body[BODY_R_LEG],
		ch->body[BODY_R_FOOT],
		ch->body[BODY_L_LEG],
		ch->body[BODY_L_FOOT],
		ch->body[BODY_CNS]
		);
		
		
	fprintf( fp, "Scars      %d %d %d %d %d %d %d %d %d %d %d %d %d %d %d %d %d\n",
	    ch->scars[BODY_NONE],
		ch->scars[BODY_R_EYE],
		ch->scars[BODY_L_EYE],
		ch->scars[BODY_HEAD],
		ch->scars[BODY_NECK],
		ch->scars[BODY_CHEST],
		ch->scars[BODY_BACK],
		ch->scars[BODY_R_ARM],
		ch->scars[BODY_R_HAND],
		ch->scars[BODY_L_ARM],
		ch->scars[BODY_L_HAND],
		ch->scars[BODY_ABDOMEN],
		ch->scars[BODY_R_LEG],
		ch->scars[BODY_R_FOOT],
		ch->scars[BODY_L_LEG],
		ch->scars[BODY_L_FOOT],
		ch->scars[BODY_CNS]
		);
		
	fprintf( fp, "Bleed     %d %d %d %d %d %d %d %d %d %d %d %d %d %d %d %d %d\n",
	    ch->bleed[BODY_NONE],
		ch->bleed[BODY_R_EYE],
		ch->bleed[BODY_L_EYE],
		ch->bleed[BODY_HEAD],
		ch->bleed[BODY_NECK],
		ch->bleed[BODY_CHEST],
		ch->bleed[BODY_BACK],
		ch->bleed[BODY_R_ARM],
		ch->bleed[BODY_R_HAND],
		ch->bleed[BODY_L_ARM],
		ch->bleed[BODY_L_HAND],
		ch->bleed[BODY_ABDOMEN],
		ch->bleed[BODY_R_LEG],
		ch->bleed[BODY_R_FOOT],
		ch->bleed[BODY_L_LEG],
		ch->bleed[BODY_L_FOOT],
		ch->bleed[BODY_CNS]
		);
		
		
	fprintf( fp, "Bandage      %d %d %d %d %d %d %d %d %d %d %d %d %d %d %d %d %d\n",
	    ch->bandage[BODY_NONE],
		ch->bandage[BODY_R_EYE],
		ch->bandage[BODY_L_EYE],
		ch->bandage[BODY_HEAD],
		ch->bandage[BODY_NECK],
		ch->bandage[BODY_CHEST],
		ch->bandage[BODY_BACK],
		ch->bandage[BODY_R_ARM],
		ch->bandage[BODY_R_HAND],
		ch->bandage[BODY_L_ARM],
		ch->bandage[BODY_L_HAND],
		ch->bandage[BODY_ABDOMEN],
		ch->bandage[BODY_R_LEG],
		ch->bandage[BODY_R_FOOT],
		ch->bandage[BODY_L_LEG],
		ch->bandage[BODY_L_FOOT],
		ch->bandage[BODY_CNS]
		);
		

	fprintf( fp, "Condition    %d %d %d\n",
	    ch->pcdata->condition[0],
	    ch->pcdata->condition[1],
	    ch->pcdata->condition[2] );
		
	fprintf( fp, "TPs    		%d %d\n",
	    ch->pcdata->PTPs,
	    ch->pcdata->MTPs );

	for ( sn = 0; sn < MAX_SKILL; sn++ )
	{
	    if ( skill_table[sn].name != NULL && ch->pcdata->learned[sn] > 0 )
	    {
		fprintf( fp, "Skill        %d '%s'\n",
		    ch->pcdata->learned[sn], skill_table[sn].name );
	    }
	}
    }

    for ( paf = ch->affected; paf != NULL; paf = paf->next )
    {
	/* Thx Alander */
	if ( paf->type < 0 || paf->type >= MAX_SPELL )
	    continue;

	fprintf( fp, "AffectData   '%s' %3d %3d %3d %10d\n",
	    spell_table[paf->type].name,
	    paf->duration,
	    paf->modifier,
	    paf->location,
	    paf->bitvector
	    );
    }

    fprintf( fp, "End\n\n" );
	
    return;
}



/*
 * Write an object and its contents.
 */
void fwrite_obj( CHAR_DATA *ch, OBJ_DATA *obj, FILE *fp, int iNest )
{
    EXTRA_DESCR_DATA *ed;
    AFFECT_DATA *paf;
	VERB_TRAP_DATA *vt;
	
    /*
     * Slick recursion to write lists backwards,
     *   so loading them will load in forwards order.
     */
    if ( obj->next_content != NULL )
	fwrite_obj( ch, obj->next_content, fp, iNest );

    fprintf( fp, "#OBJECT\n" );
    fprintf( fp, "Nest         %d\n",	iNest			     );
    fprintf( fp, "Name         %s~\n",	obj->name		     );
    fprintf( fp, "ShortDescr   %s~\n",	obj->short_descr	     );
    fprintf( fp, "Description  %s~\n",	obj->description	     );
    fprintf( fp, "Vnum         %d\n",	obj->pIndexData->vnum	     );
    fprintf( fp, "ExtraFlags   %d\n",	obj->extra_flags	     );
    fprintf( fp, "WearFlags    %ld\n",	obj->wear_flags		     );
    fprintf( fp, "WearLoc      %d\n",	obj->wear_loc		     );
    fprintf( fp, "ItemType     %d\n",	obj->item_type		     );
    fprintf( fp, "Weight       %d\n",	obj->weight		     );
    fprintf( fp, "Level        %d\n",	obj->level		     );
    fprintf( fp, "Timer        %d\n",	obj->timer		     );
    fprintf( fp, "Cost         %d\n",	obj->cost		     );
	
    fprintf( fp, "Values       %d %d %d %d\n",
	obj->value[0], obj->value[1], obj->value[2], obj->value[3]	     );
	
	if ( obj->obj_spec_name != NULL )
	fprintf( fp, "Special         %s~\n",	obj->obj_spec_name	     );	

	fprintf( fp, "ST           %d\n",	obj->st	);
	fprintf( fp, "DU           %d\n",	obj->du	 );


    switch ( obj->item_type )
    {
    case ITEM_POTION:
    case ITEM_SCROLL:
	if ( obj->value[1] > 0 )
	{
	    fprintf( fp, "Spell 1      '%s'\n",
		spell_table[obj->value[1]].name );
	}

	if ( obj->value[2] > 0 )
	{
	    fprintf( fp, "Spell 2      '%s'\n",
		spell_table[obj->value[2]].name );
	}

	if ( obj->value[3] > 0 )
	{
	    fprintf( fp, "Spell 3      '%s'\n",
		spell_table[obj->value[3]].name );
	}

	break;

    case ITEM_PILL:
    case ITEM_STAFF:
    case ITEM_WAND:
	if ( obj->value[3] > 0 )
	{
	    fprintf( fp, "Spell 3      '%s'\n",
		spell_table[obj->value[3]].name );
	}

	break;
    }

    for ( paf = obj->affected; paf != NULL; paf = paf->next )
    {
	/*
	if ( paf->type < 0 || paf->type >= MAX_SPELL )
	    continue;
	*/

	fprintf( fp, "AffectData   '%s' %d %d %d %d\n",
		(paf->type > 0 && paf->type < MAX_SPELL ?
		spell_table[paf->type].name :
		"obj" ),
	    paf->duration,
	    paf->modifier,
	    paf->location,
	    paf->bitvector
	    );
    }

    for ( ed = obj->extra_descr; ed != NULL; ed = ed->next )
    {
	fprintf( fp, "ExtraDescr   %s~ %s~\n",
	    ed->keyword, ed->description );
    }
	
	if( obj->verb_trap != NULL )
	{
		printf("saving %s's verb trap %s\r\n", obj->short_descr, obj->verb_trap->verb );
	
	for ( vt = obj->verb_trap; vt != NULL; vt = vt->next )
    {
	fprintf( fp, "VerbTrap   %s~ %s~ %s~ %s~\n", vt->verb, vt->firstPersonMessage, vt->roomMessage, vt->GSL );
    }
	}
	
    fprintf( fp, "End\n\n" );
	
    if ( obj->contains != NULL )
	fwrite_obj( ch, obj->contains, fp, iNest + 1 );

    return;
}



/*
 * Load a char and inventory into a new ch structure.
 */
bool load_char_obj( DESCRIPTOR_DATA *d, char *name )
{
    static PC_DATA pcdata_zero;
    char strsave[MAX_INPUT_LENGTH];
    CHAR_DATA *ch;
    FILE *fp;
    bool found;
	int iBody = 1;

    if ( char_free == NULL )
    {
	ch				= alloc_perm( sizeof(*ch) );
    }
    else
    {
	ch				= char_free;
	char_free			= char_free->next;
    }
    clear_char( ch );

    if ( pcdata_free == NULL )
    {
	ch->pcdata			= alloc_perm( sizeof(*ch->pcdata) );
    }
    else
    {
	ch->pcdata			= pcdata_free;
	pcdata_free			= pcdata_free->next;
    }
    *ch->pcdata				= pcdata_zero;

    d->character			= ch;
    ch->desc				= d;
    ch->name				= str_dup( name );
    ch->act				= 0;
    ch->pcdata->pwd			= str_dup( "" );
    ch->pcdata->bamfin			= str_dup( "" );
    ch->pcdata->bamfout			= str_dup( "" );
    ch->pcdata->title			= str_dup( "" );
    ch->pcdata->perm_Strength		= 50;
    ch->pcdata->perm_Endurance		= 50;
	ch->pcdata->perm_Dexterity		= 50;
	ch->pcdata->perm_Speed			= 50;
	ch->pcdata->perm_Willpower		= 50;
	ch->pcdata->perm_Potency		= 50;
    ch->pcdata->perm_Judgement		= 50;
	ch->pcdata->perm_Intelligence	= 50;
	ch->pcdata->perm_Wisdom			= 50;
	ch->pcdata->perm_Charm			= 50;
    ch->pcdata->condition[COND_THIRST]	= 48;
    ch->pcdata->condition[COND_FULL]	= 48;
	ch->pcdata->PTPs					= 0;
	ch->pcdata->MTPs					= 0;
	ch->as = 0;
	ch->ds = 0;
	ch->cs = 0;
	ch->td = 0;
	ch->decay = 0;
	
	while( iBody < MAX_BODY )
	{
	ch->body[iBody] = 0;
	iBody++;
	}
	
	iBody = 0;
	
	while( iBody < MAX_BODY )
	{
	ch->scars[iBody] = 0;
	iBody++;
	}
	
	iBody = 0;
	
	while( iBody < MAX_BODY )
	{
	ch->bleed[iBody] = 0;
	iBody++;
	}
	
	iBody = 0;
	
	while( iBody < MAX_BODY )
	{
	ch->bandage[iBody] = 0;
	iBody++;
	}


    found = FALSE;
    fclose( fpReserve );
    sprintf( strsave, "%s%s", PLAYER_DIR, capitalize( name ) );
    if ( ( fp = fopen( strsave, "r" ) ) != NULL )
    {
	int iNest;

	for ( iNest = 0; iNest < MAX_NEST; iNest++ )
	    rgObjNest[iNest] = NULL;

	found = TRUE;
	for ( ; ; )
	{
	    char letter;
	    char *word;

	    letter = fread_letter( fp );
	    if ( letter == '*' )
	    {
		fread_to_eol( fp );
		continue;
	    }

	    if ( letter != '#' )
	    {
		bug( "Load_char_obj: # not found.", 0 );
		break;
	    }

	    word = fread_word( fp );
	    if      ( !str_cmp( word, "PLAYER" ) ) fread_char ( ch, fp );
	    else if ( !str_cmp( word, "OBJECT" ) ) fread_obj  ( ch, fp );
	    else if ( !str_cmp( word, "END"    ) ) break;
	    else
	    {
		bug( "Load_char_obj: bad section.", 0 );
		break;
	    }
	}
	fclose( fp );
    }

    fpReserve = fopen( NULL_FILE, "r" );
    return found;
}



/*
 * Read in a char.
 */

#if defined(KEY)
#undef KEY
#endif

#define KEY( literal, field, value )					\
				if ( !str_cmp( word, literal ) )	\
				{					\
				    field  = value;			\
				    fMatch = TRUE;			\
				    break;				\
				}

void fread_char( CHAR_DATA *ch, FILE *fp )
{
    char buf[MAX_STRING_LENGTH];
    char *word;
    bool fMatch;

    for ( ; ; )
    {
	word   = feof( fp ) ? "End" : fread_word( fp );
	fMatch = FALSE;

	switch ( UPPER(word[0]) )
	{
	case '*':
	    fMatch = TRUE;
	    fread_to_eol( fp );
	    break;

	case 'A':
	    KEY( "Act",		ch->act,		fread_number( fp ) );
	    KEY( "AffectedBy",	ch->affected_by,	fread_number( fp ) );
		KEY( "Age", ch->pcdata->age, fread_number( fp ) );
	    KEY( "Armor",	ch->armor,		fread_number( fp ) );

	    if ( !str_cmp( word, "Affect" ) || !str_cmp( word, "AffectData" ) )
	    {
		AFFECT_DATA *paf;

		if ( affect_free == NULL )
		{
		    paf		= alloc_perm( sizeof(*paf) );
		}
		else
		{
		    paf		= affect_free;
		    affect_free	= affect_free->next;
		}

		if ( !str_cmp( word, "Affect" ) )
		{
		    /* Obsolete 2.0 form. */
		    paf->type	= fread_number( fp );
		}
		else
		{
		    int sn;
			
		    sn = spell_lookup( fread_word( fp ) );
		    if ( sn < 0 )
			bug( "Fread_char: unknown spell.", 0 );
		    else 
			
			paf->type = sn;
		}

		paf->duration	= fread_number( fp );
		paf->modifier	= fread_number( fp );
		paf->location	= fread_number( fp );
		paf->bitvector	= fread_number( fp );
		paf->next	= ch->affected;
		ch->affected	= paf;
		fMatch = TRUE;
		break;
	    }

	    if ( !str_cmp( word, "AttrMod"  ) )
	    {
		ch->pcdata->mod_Strength		= fread_number( fp );
		ch->pcdata->mod_Endurance		= fread_number( fp );
		ch->pcdata->mod_Dexterity		= fread_number( fp );
		ch->pcdata->mod_Speed			= fread_number( fp );
		ch->pcdata->mod_Willpower		= fread_number( fp );
		ch->pcdata->mod_Potency			= fread_number( fp );
		ch->pcdata->mod_Judgement		= fread_number( fp );
		ch->pcdata->mod_Intelligence	= fread_number( fp );
		ch->pcdata->mod_Wisdom			= fread_number( fp );
		ch->pcdata->mod_Charm			= fread_number( fp );
		fMatch = TRUE;
		break;
	    }

	    if ( !str_cmp( word, "AttrPerm" ) )
	    {
		ch->pcdata->perm_Strength		= fread_number( fp );
		ch->pcdata->perm_Endurance		= fread_number( fp );
		ch->pcdata->perm_Dexterity		= fread_number( fp );
		ch->pcdata->perm_Speed			= fread_number( fp );
		ch->pcdata->perm_Willpower		= fread_number( fp );
		ch->pcdata->perm_Potency		= fread_number( fp );
		ch->pcdata->perm_Judgement		= fread_number( fp );
		ch->pcdata->perm_Intelligence	= fread_number( fp );
		ch->pcdata->perm_Wisdom			= fread_number( fp );
		ch->pcdata->perm_Charm			= fread_number( fp );
		fMatch = TRUE;
		break;
	    }
	    break;

	case 'B':
	    KEY( "Bamfin",	ch->pcdata->bamfin,	fread_string( fp ) );
	    KEY( "Bamfout",	ch->pcdata->bamfout,	fread_string( fp ) );
		KEY( "Bank",	ch->pcdata->bank,	fread_number( fp ) );
		
		if ( !str_cmp( word, "Bandage" ) )
	    {
		ch->bandage[BODY_NONE]				= fread_number( fp );
		ch->bandage[BODY_R_EYE]		= fread_number( fp );
		ch->bandage[BODY_L_EYE]		= fread_number( fp );
		ch->bandage[BODY_HEAD]		= fread_number( fp );
		ch->bandage[BODY_NECK]		= fread_number( fp );
		ch->bandage[BODY_CHEST]		= fread_number( fp );
		ch->bandage[BODY_BACK]		= fread_number( fp );
		ch->bandage[BODY_R_ARM]		= fread_number( fp );
		ch->bandage[BODY_R_HAND]			= fread_number( fp );
		ch->bandage[BODY_L_ARM]		= fread_number( fp );
		ch->bandage[BODY_L_HAND]			= fread_number( fp );
		ch->bandage[BODY_ABDOMEN]		= fread_number( fp );
		ch->bandage[BODY_R_LEG]			= fread_number( fp );
		ch->bandage[BODY_R_FOOT]		= fread_number( fp );
		ch->bandage[BODY_L_LEG]			= fread_number( fp );
		ch->bandage[BODY_L_FOOT]			= fread_number( fp );
		ch->bandage[BODY_CNS]			= fread_number( fp );
		fMatch = TRUE;
		break;
	    }
		
		if ( !str_cmp( word, "BaseStats" ) )
	    {
		ch->as	= fread_number( fp );
		ch->ds	= fread_number( fp );
		ch->cs	= fread_number( fp );
		ch->td	= fread_number( fp );
		fMatch = TRUE;
		break;
	    }
		
		if ( !str_cmp( word, "Body" ) )
	    {
		ch->body[BODY_NONE]				= fread_number( fp );
		ch->body[BODY_R_EYE]		= fread_number( fp );
		ch->body[BODY_L_EYE]		= fread_number( fp );
		ch->body[BODY_HEAD]		= fread_number( fp );
		ch->body[BODY_NECK]		= fread_number( fp );
		ch->body[BODY_CHEST]		= fread_number( fp );
		ch->body[BODY_BACK]		= fread_number( fp );
		ch->body[BODY_R_ARM]		= fread_number( fp );
		ch->body[BODY_R_HAND]			= fread_number( fp );
		ch->body[BODY_L_ARM]		= fread_number( fp );
		ch->body[BODY_L_HAND]			= fread_number( fp );
		ch->body[BODY_ABDOMEN]		= fread_number( fp );
		ch->body[BODY_R_LEG]			= fread_number( fp );
		ch->body[BODY_R_FOOT]		= fread_number( fp );
		ch->body[BODY_L_LEG]			= fread_number( fp );
		ch->body[BODY_L_FOOT]			= fread_number( fp );
		ch->body[BODY_CNS]			= fread_number( fp );
		fMatch = TRUE;
		break;
	    }
		
		if ( !str_cmp( word, "Bleed" ) )
	    {
		ch->bleed[BODY_NONE]				= fread_number( fp );
		ch->bleed[BODY_R_EYE]		= fread_number( fp );
		ch->bleed[BODY_L_EYE]		= fread_number( fp );
		ch->bleed[BODY_HEAD]		= fread_number( fp );
		ch->bleed[BODY_NECK]		= fread_number( fp );
		ch->bleed[BODY_CHEST]		= fread_number( fp );
		ch->bleed[BODY_BACK]		= fread_number( fp );
		ch->bleed[BODY_R_ARM]		= fread_number( fp );
		ch->bleed[BODY_R_HAND]			= fread_number( fp );
		ch->bleed[BODY_L_ARM]		= fread_number( fp );
		ch->bleed[BODY_L_HAND]			= fread_number( fp );
		ch->bleed[BODY_ABDOMEN]		= fread_number( fp );
		ch->bleed[BODY_R_LEG]			= fread_number( fp );
		ch->bleed[BODY_R_FOOT]		= fread_number( fp );
		ch->bleed[BODY_L_LEG]			= fread_number( fp );
		ch->bleed[BODY_L_FOOT]			= fread_number( fp );
		ch->bleed[BODY_CNS]			= fread_number( fp );
		fMatch = TRUE;
		break;
	    }
	    break;

	case 'C':
	    KEY( "Class",	ch->class,		fread_number( fp ) );

	    if ( !str_cmp( word, "Condition" ) )
	    {
		ch->pcdata->condition[0] = fread_number( fp );
		ch->pcdata->condition[1] = fread_number( fp );
		ch->pcdata->condition[2] = fread_number( fp );
		fMatch = TRUE;
		break;
	    }
	    break;

	case 'D':
	    KEY( "Damroll",	ch->damroll,		fread_number( fp ) );
	    KEY( "Deaf",	ch->deaf,		fread_number( fp ) );
		KEY( "Decay",	ch->decay,		fread_number( fp ) );
		KEY( "Deeds",	ch->pcdata->deeds,		fread_number( fp ) );
	    KEY( "Description",	ch->description,	fread_string( fp ) );
	    break;

	case 'E':
	    if ( !str_cmp( word, "End" ) )
			return;
		
		KEY( "Eyes", 	ch->pcdata->eyes, fread_string(fp ) );
	    KEY( "Exp",		ch->exp,		fread_number( fp ) );
	    break;
		
	case 'F':
	    KEY( "FieldExp",ch->pcdata->fieldExp,		fread_number( fp ) );
	    break;

	case 'G':
	    KEY( "Gold",	ch->gold,		fread_number( fp ) );
	    break;

	case 'H':
		KEY( "Hair", ch->pcdata->hair, fread_string( fp ) );
	    KEY( "Hitroll",	ch->hitroll,		fread_number( fp ) );

	    if ( !str_cmp( word, "HpManaMove" ) )
	    {
		ch->hit		= fread_number( fp );
		ch->max_hit	= fread_number( fp );
		ch->mana	= fread_number( fp );
		ch->max_mana	= fread_number( fp );
		ch->move	= fread_number( fp );
		ch->max_move	= fread_number( fp );
		fMatch = TRUE;
		break;
	    }
	    break;

      case 'I':
         break;

	case 'L':
	    KEY( "Level",	ch->level,		fread_number( fp ) );
	    KEY( "LongDescr",	ch->long_descr,		fread_string( fp ) );
	    break;

	case 'N':
	    if ( !str_cmp( word, "Name" ) )
	    {
		/*
		 * Name already set externally.
		 */
		fread_to_eol( fp );
		fMatch = TRUE;
		break;
	    }

	    break;

	case 'P':
	    KEY( "Password",	ch->pcdata->pwd,	fread_string( fp ) );
	    KEY( "Played",	ch->played,		fread_number( fp ) );
	    KEY( "Position",	ch->position,		fread_number( fp ) );
	    KEY( "Practice",	ch->practice,		fread_number( fp ) );
	    break;

	case 'R':
	    KEY( "Race",        ch->race,		fread_number( fp ) );

	    if ( !str_cmp( word, "Room" ) )
	    {
		ch->in_room = get_room_index( fread_number( fp ) );
		if ( ch->in_room == NULL )
		    ch->in_room = get_room_index( ROOM_VNUM_LIMBO );
		fMatch = TRUE;
		break;
	    }

	    break;

	case 'S':
	    KEY( "SavingThrow",	ch->saving_throw,	fread_number( fp ) );
	    KEY( "Sex",		ch->sex,		fread_number( fp ) );
		KEY( "Skin", ch->pcdata->skin, fread_string( fp ) );
		
	    KEY( "ShortDescr",	ch->short_descr,	fread_string( fp ) );
	    KEY( "Stance",		ch->stance,		fread_number( fp ) );
		
	    if ( !str_cmp( word, "Skill" ) )
	    {
		int sn;
		int value;

		value = fread_number( fp );
		sn    = skill_lookup( fread_word( fp ) );
		if ( sn < 0 )
		    bug( "Fread_word: unknown skill.", 0 );
		else
		    ch->pcdata->learned[sn] = value;
		fMatch = TRUE;
	    }
		
		
		if ( !str_cmp( word, "Scars" ) )
	    {
		ch->scars[BODY_NONE]				= fread_number( fp );
		ch->scars[BODY_R_EYE]		= fread_number( fp );
		ch->scars[BODY_L_EYE]		= fread_number( fp );
		ch->scars[BODY_HEAD]		= fread_number( fp );
		ch->scars[BODY_NECK]		= fread_number( fp );
		ch->scars[BODY_CHEST]		= fread_number( fp );
		ch->scars[BODY_BACK]		= fread_number( fp );
		ch->scars[BODY_R_ARM]		= fread_number( fp );
		ch->scars[BODY_R_HAND]			= fread_number( fp );
		ch->scars[BODY_L_ARM]		= fread_number( fp );
		ch->scars[BODY_L_HAND]			= fread_number( fp );
		ch->scars[BODY_ABDOMEN]		= fread_number( fp );
		ch->scars[BODY_R_LEG]			= fread_number( fp );
		ch->scars[BODY_R_FOOT]		= fread_number( fp );
		ch->scars[BODY_L_LEG]			= fread_number( fp );
		ch->scars[BODY_L_FOOT]			= fread_number( fp );
		ch->scars[BODY_CNS]			= fread_number( fp );
		fMatch = TRUE;
		break;
	    }

	    break;

	case 'T':
	    KEY( "Trust",	ch->trust,		fread_number( fp ) );

	    if ( !str_cmp( word, "Title" ) )
	    {
		ch->pcdata->title = fread_string( fp );
		if ( isalpha((unsigned char) ch->pcdata->title[0])
		||   isdigit((unsigned char) ch->pcdata->title[0]) )
		{
		    sprintf( buf, " %s", ch->pcdata->title );
		    free_string( ch->pcdata->title );
		    ch->pcdata->title = str_dup( buf );
		}
		fMatch = TRUE;
		break;
	    }
		
		if ( !str_cmp( word, "TPs" ) )
	    {
		ch->pcdata->PTPs		= fread_number( fp );
		ch->pcdata->MTPs		= fread_number( fp );
		fMatch = TRUE;
		break;
	    }

	    break;

	case 'V':
	    if ( !str_cmp( word, "Vnum" ) )
	    {
		ch->pIndexData = get_mob_index( fread_number( fp ) );
		fMatch = TRUE;
		break;
	    }
	    break;
		
	case 'W':
	    if ( !str_cmp( word, "Wrap" ) )
	    {
		ch->pcdata->wrap = fread_number( fp ) ;
		fMatch = TRUE;
		break;
	    }
	    break;
	}

	if ( !fMatch )
	{
	    bug( "Fread_char: no match.", 0 );
		printf("stuck on %s\r\n", word );
	    fread_to_eol( fp );
	}
    }
}



void fread_obj( CHAR_DATA *ch, FILE *fp )
{
    static OBJ_DATA obj_zero;
    OBJ_DATA *obj;
    char *word;
    int iNest;
    bool fMatch;
    bool fNest;
    bool fVnum;

    if ( obj_free == NULL )
    {
	obj		= alloc_perm( sizeof(*obj) );
    }
    else
    {
	obj		= obj_free;
	obj_free	= obj_free->next;
    }

    *obj		= obj_zero;
    obj->name		= str_dup( "" );
    obj->short_descr	= str_dup( "" );
    obj->description	= str_dup( "" );

    fNest		= FALSE;
    fVnum		= TRUE;
    iNest		= 0;

    for ( ; ; )
    {
	word   = feof( fp ) ? "End" : fread_word( fp );
	fMatch = FALSE;

	switch ( UPPER(word[0]) )
	{
	case '*':
	    fMatch = TRUE;
	    fread_to_eol( fp );
	    break;

	case 'A':
	    if ( !str_cmp( word, "Affect" ) || !str_cmp( word, "AffectData" ) )
	    {
		AFFECT_DATA *paf;

		if ( affect_free == NULL )
		{
		    paf		= alloc_perm( sizeof(*paf) );
		}
		else
		{
		    paf		= affect_free;
		    affect_free	= affect_free->next;
		}

		if ( !str_cmp( word, "Affect" ) )
		{
		    /* Obsolete 2.0 form. */
		    paf->type	= fread_number( fp );
		}
		else
		{
		    int sn = -1;
			
			fread_word(fp);
			/*
		    sn = spell_lookup( fread_word( fp ) );
		    if ( sn < 0 )
			bug( "Fread_char: unknown spell.", 0 );
		    else */
			
			paf->type = sn;
		}

		//paf->type	= fread_number( fp );
		paf->duration	= fread_number( fp );
		paf->modifier	= fread_number( fp );
		paf->location	= fread_number( fp );
		paf->bitvector	= fread_number( fp );
		paf->next	= obj->affected;
		obj->affected	= paf;
		fMatch		= TRUE;
		break;
	    }
	    break;

	case 'C':
	    KEY( "Cost",	obj->cost,		fread_number( fp ) );
	    break;

	case 'D':
	    KEY( "Description",	obj->description,	fread_string( fp ) );
		KEY( "DU",	obj->du,		fread_number( fp ) );
	    
	    break;

	case 'E':
	    KEY( "ExtraFlags",	obj->extra_flags,	fread_number( fp ) );

	    if ( !str_cmp( word, "ExtraDescr" ) )
	    {
		EXTRA_DESCR_DATA *ed;

		if ( extra_descr_free == NULL )
		{
		    ed			= alloc_perm( sizeof(*ed) );
		}
		else
		{
		    ed			= extra_descr_free;
		    extra_descr_free	= extra_descr_free->next;
		}

		ed->keyword		= fread_string( fp );
		ed->description		= fread_string( fp );
		ed->next		= obj->extra_descr;
		obj->extra_descr	= ed;
		fMatch = TRUE;
	    }

	    if ( !str_cmp( word, "End" ) )
	    {
		if ( !fNest || !fVnum )
		{
		    bug( "Fread_obj: incomplete object.", 0 );
		/*  free_string( obj->name        );
		    free_string( obj->description );
		    free_string( obj->short_descr );
		    obj->next = obj_free;
		    obj_free  = obj;
			*/
			
			extract_obj(obj);
		    return;
		}
		else
		{
		    obj->next	= object_list;
		    object_list	= obj;
		    obj->pIndexData->count++;
		    if ( iNest == 0 || rgObjNest[iNest] == NULL )
			obj_to_char( obj, ch );
		    else
			obj_to_obj( obj, rgObjNest[iNest-1] );
		    return;
		}
	    }
	    break;

	case 'I':
	    KEY( "ItemType",	obj->item_type,		fread_number( fp ) );
	    break;

	case 'L':
	    KEY( "Level",	obj->level,		fread_number( fp ) );
	    break;

	case 'N':
	    KEY( "Name",	obj->name,		fread_string( fp ) );

	    if ( !str_cmp( word, "Nest" ) )
	    {
		iNest = fread_number( fp );
		if ( iNest < 0 || iNest >= MAX_NEST )
		{
		    bug( "Fread_obj: bad nest %d.", iNest );
		}
		else
		{
		    rgObjNest[iNest] = obj;
		    fNest = TRUE;
		}
		fMatch = TRUE;
	    }
	    break;

	case 'S':
	    KEY( "ShortDescr",	obj->short_descr,	fread_string( fp ) );
		KEY( "ST",	obj->st,		fread_number( fp ) );
		
		if ( !str_cmp( word, "Special" ) )
	    {
		obj->obj_spec_name = fread_string( fp );
		obj->spec_fun = obj_spec_lookup( obj->obj_spec_name );
		fMatch = TRUE;
		break;
	    }
		
	    if ( !str_cmp( word, "Spell" ) )
	    {
		int iValue;
		int sn;

		iValue = fread_number( fp );
		sn     = spell_lookup( fread_word( fp ) );
		if ( iValue < 0 || iValue > 3 )
		{
		    bug( "Fread_obj: bad iValue %d.", iValue );
		}
		else if ( sn < 0 )
		{
		    bug( "Fread_obj: unknown skill.", 0 );
		}
		else
		{
		    obj->value[iValue] = sn;
		}
		fMatch = TRUE;
		break;
	    }
	
	    break;

	case 'T':
	    KEY( "Timer",	obj->timer,		fread_number( fp ) );
	    break;

	case 'V':
	    if ( !str_cmp( word, "Values" ) )
	    {
		obj->value[0]	= fread_number( fp );
		obj->value[1]	= fread_number( fp );
		obj->value[2]	= fread_number( fp );
		obj->value[3]	= fread_number( fp );
		fMatch		= TRUE;
		break;
	    }
		
	    if ( !str_cmp( word, "VerbTrap" ) )
	    {
		VERB_TRAP_DATA *vt;

		if ( verb_trap_free == NULL )
		{
		    vt			= alloc_perm( sizeof(*vt) );
		}
		else
		{
		    vt				= verb_trap_free;
		    verb_trap_free	= verb_trap_free->next;
		}

		vt->verb					= fread_string( fp );
		vt->firstPersonMessage		= fread_string( fp );
		vt->roomMessage				= fread_string( fp );
		vt->GSL						= fread_string( fp );
		
		vt->next = obj->verb_trap;
		obj->verb_trap	= vt;

		fMatch = TRUE;
	    }

	    if ( !str_cmp( word, "Vnum" ) )
	    {
		int vnum;

		vnum = fread_number( fp );
		if ( ( obj->pIndexData = get_obj_index( vnum ) ) == NULL )
		    bug( "Fread_obj: bad vnum %d.", vnum );
		else
		    fVnum = TRUE;
		fMatch = TRUE;
		break;
	    }
	    break;

	case 'W':
	    KEY( "WearFlags",	obj->wear_flags,	fread_number( fp ) );
	    KEY( "WearLoc",	obj->wear_loc,		fread_number( fp ) );
	    KEY( "Weight",	obj->weight,		fread_number( fp ) );
	    break;

	}

	if ( !fMatch )
	{
	    bug( "Fread_obj: no match.", 0 );
	    fread_to_eol( fp );
	}
    }
}
//specials for mobs, rooms, and objs.

#if defined(macintosh)
#include <types.h>
#else
#include <sys/types.h>
#endif
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <time.h>
#include "merc.h"






/*
 * Given a name, return the appropriate spec fun.
 */


ROOM_SPECIAL *room_fun_lookup( const char *name )
{
    if ( !str_cmp( name, "spec_start_room"       ) ) return spec_start_room;
    if ( !str_cmp( name, "spec_orchard"          ) ) return spec_orchard;
	if ( !str_cmp( name, "spec_temple"          ) ) return spec_temple;
    return 0;
}



OBJ_FUN *obj_spec_lookup( const char *name )
{
    if ( !str_cmp( name, "spec_armsheath"          ) ) return spec_armsheath;
	if ( !str_cmp( name, "spec_trashbin"          ) ) return spec_trashbin;
    if ( !str_cmp( name, "spec_flame_blade"        ) ) return spec_flame_blade;
	if ( !str_cmp( name, "spec_glow"        ) ) return spec_glow;
    return 0;
}


SPEC_FUN *spec_lookup( const char *name )
{
    if ( !str_cmp( name, "spec_healer"		  ) ) return spec_healer;
    return 0;
}

//fixme add rt
//fixme add debts

bool is_hurt( CHAR_DATA *ch )
{
	short x;
	
	for( x = 0; x < MAX_BODY; x++ )
	{
	if( ch->body[x] != 0 || ch->scars[x] != 0 || ch->bleed[x] != 0 )
		return TRUE;
	}
	
	return FALSE;
}

//bool spec_healer( CHAR_DATA *ch ) -- currently an issue with healing hp
bool spec_healer( CHAR_DATA *ch, char *cmd, DO_FUN *fnptr, char *arg, CHAR_DATA *mob )
{
    CHAR_DATA *victim;
    CHAR_DATA *v_next;
	short x;
 
    if ( mob->position != POS_STANDING )
		return FALSE;
		
    for ( victim = mob->in_room->people; victim != NULL; victim = v_next )
    {
	v_next = victim->next_in_room;
 
	if ( IS_NPC(victim) || !can_see( mob, victim ) || !is_hurt(victim) || mob->hunting == victim )	
	    continue;
	
	if(	victim->wait > 0 )
	return FALSE;
	
	if ( victim->position != POS_PRONE  )
	{
	
	if( victim->hit < victim->max_hit )
	{
	act( "$n wraps your wounds with clean linen bandages.", mob, NULL, victim, TO_VICT );
	act( "Tendrils reach out of $n's hands, encompassing $N's wounds, which disappear.", mob, NULL, victim, TO_NOTVICT );
	do_say( mob, "You are now well.  Lie down so I can treat you further, or begone." );
	victim->hit = victim->max_hit;
	mob->wait += 1;
	}

	add_roundtime( mob, dice(2,6) );
	return FALSE;
	}
 
	for( x = 0; x < MAX_BODY; x++ )
	{
		if( victim->bleed[x] > 0 && victim->bandage[x] == 0 )
		{
			printf_to_char(mob, "You bandage the %s.\r\n", body_name[x] );
			act( "$n wraps $N's wounds with clean linen bandages.", mob, NULL, victim, TO_NOTVICT );
			act( "$n wraps your wounds with clean linen bandages.", mob, NULL, victim, TO_VICT );
			victim->bandage[x] = 100;
			add_roundtime( victim, 30 );
			show_roundtime( victim, 30 );
			return TRUE;
		}
	}
	
	for( x = 0; x < MAX_BODY; x++ )
	{
		if( victim->body[x] > 0 )
		{
			printf_to_char(mob, "You heal the %s.\r\n", body_name[x] );
			act( "$n applies a sweet-smelling herbal poultice to $N's wounds, healing them.", mob, NULL, victim, TO_NOTVICT );
			act( "$n applies a sweet-smelling herbal poultice to your wounds, healing them.", mob, NULL, victim, TO_VICT );
			victim->scars[x] = victim->body[x];
			victim->body[x] = 0;
			victim->bleed[x] = 0;
			victim->bandage[x] = 0;
			add_roundtime( victim, 30 );
			show_roundtime( victim, 30 );
			return TRUE;
		}
	}
	
	for( x = 0; x < MAX_BODY; x++ )
	{
		if( victim->scars[x] > 0 )
		{
			printf_to_char(mob, "You cure the scars on %s.\r\n", body_name[x] );
			act( "$n applies a thick white ointment to $N's scars, which causes them to fade away.", mob, NULL, victim, TO_NOTVICT );
			act( "$n applies a thick white ointment to your scars, which causes them to fade away.", mob, NULL, victim, TO_VICT );
			victim->scars[x] = 0;
			add_roundtime( victim, 30 );
			show_roundtime( victim, 30 );
			return TRUE;
		}
	}
	
	if( victim->hit < victim->max_hit )
	{
		printf_to_char(mob, "You heal %s's blood loss.\r\n", victim->name );
		act( "$n force $N to drink a bright red potion.  $N looks invigorated.", mob, NULL, victim, TO_NOTVICT );
		act( "$n forces you drink a bright red potion, restoring your health.", mob, NULL, victim, TO_VICT );
		victim->hit = victim->max_hit;
		add_roundtime( victim, 30 );
		show_roundtime( victim, 30 );
		return TRUE;
	}	
 
 
    }
 
    return FALSE;
}



bool spec_temple( CHAR_DATA *ch, char *command, DO_FUN *fnptr, char *argument, ROOM_INDEX_DATA *room )
{
    if (IS_NPC(ch))
	return FALSE;

    if ( !str_cmp( command, "donate" ) )
    {
	if ( argument == NULL )
	{
	    send_to_char("Donate how much?\n\r", ch );
	    return TRUE;
	}
	else 
    {
	/* 'drop NNNN coins' */
	int amount;

	amount   = atoi(argument );

	if ( amount < 1 )
	{
		printf_to_char( ch, "You reconsider your paradoxical donation of %d coins.\n\r", amount );
	    return TRUE;
	}

	if ( ch->gold < amount )
	{
	    send_to_char( "You haven't got that many coins.\r\n", ch );
	    return TRUE;
	}

	ch->gold -= amount;
	
	if( amount > 1 )
	act( "$n drops some coins into a slot in one of the temple's pillars.", ch, NULL, NULL, TO_ROOM );
	else
	act( "$n drops a coin into a slot in one of the temple's pillars.", ch, NULL, NULL, TO_ROOM );
	
	printf_to_char( ch, "You drop %d coin%s into a slot in one of the temple's pillars.\r\n", amount, amount>1?"s":"");
	if( !IS_NPC(ch) && amount > ( ( ch->pcdata->deeds+1) * 250 ) )
	{
		ch->pcdata->deeds++;
		printf_to_char( ch, "You feel a sense of peace as the temple's candles burn slightly brighter.\r\n" );
		act( "The candles set around the temple blaze brightly!", ch, NULL, NULL, TO_ROOM );
	}
	else
	{
		printf_to_char( ch, "You feel hollow and empty as the temple's candles dim ever so slightly.\r\n" );
		act( "The candles set around the temple dim slightly.", ch, NULL, NULL, TO_ROOM );
	}
	}   

	return TRUE;
    }

    return FALSE;
}


bool spec_start_room( CHAR_DATA *ch, char *command, DO_FUN *fnptr,
  char *argument, ROOM_INDEX_DATA *room )
{
    if (IS_NPC(ch))
		return FALSE;

	if( !is_name(command, "help") 
		&& !is_name(command, "quit") 
		&& !is_name(command, "roll") 
		&& !is_name(command, "change") 
		&& !is_name(command, "select") 
		&& !is_name(command, "begin") 
		&& !is_name(command, "age") 
		&& !is_name(command, "eyes") 
		&& !is_name(command, "hair") 
		&& !is_name(command, "skin")
		&& !is_name(command, "set") )
	{
	printf_to_char(ch, "You suddenly know '%s' cannot be done, as a voice recites calmy:\r\n\r\nSELECT your race\r\nROLL   your stats\r\nCHANGE one stat with another\r\nSET    game settings\r\nBEGIN  the game!\r\n", command );
	return TRUE;
	}
	
	
	
    return FALSE;
}




bool spec_orchard( CHAR_DATA *ch, char *command, DO_FUN *fnptr,
 char *argument, ROOM_INDEX_DATA *room )
{
    //OBJ_DATA  *obj;

    /* if they didn't type pick, we dont care. */
    if ( str_cmp( command, "pick" ) )
	return FALSE;

    send_to_char("Need test for membership in apropriate profession here.\n\r", ch );

/*
    if ( !str_cmp( argument, "apple" )
    && is_name( "apple", ch->in_room->name ) )
    {
	if ( is_name( "green", ch->in_room->name ) )
	    obj = create_object( get_obj_index( OBJ_VNUM_GREEN_APPLE ) );
	else
	    obj = create_object( get_obj_index( OBJ_VNUM_RED_APPLE ) );

	obj_to_char( obj, ch );

	act("$n picks $p from a nearby tree.", ch, NULL, obj, NULL, NULL, TO_ROOM, SENSE_SIGHT );
	act("You pick $p from a nearby tree.", ch, NULL, obj, NULL, NULL, TO_CHAR, SENSE_SIXTH );

	return TRUE;
    }

    if ( !str_cmp( argument, "grapes" )
    && ( is_name( "vinyard", ch->in_room->name )
	|| is_name( "vinyards", ch->in_room->name ) ) )
    {
	obj = create_object( get_obj_index( OBJ_VNUM_RED_GRAPES ) );

	obj_to_char( obj, ch );

	act("$n picks $p from a vine.", ch, NULL, obj, NULL, NULL, TO_ROOM, SENSE_SIGHT );
	act("You pick $p from a vine.", ch, NULL, obj, NULL, NULL, TO_CHAR, SENSE_SIXTH );

	return TRUE;
    }
	*/

    return FALSE;
}



bool spec_armsheath( CHAR_DATA *ch, char *command, DO_FUN *fnptr, char *argument, OBJ_DATA *obj )
{
    //char arg[MAX_STRING_LENGTH];
    //char arg2[MAX_STRING_LENGTH];
    //OBJ_DATA *weapon;
	
	printf("armsheath called\r\n");
	
	/*

    if ( obj->wear_loc == gn_wear_none )
	return FALSE;

    if ( fnptr != do_draw && fnptr != do_sheath )
	return FALSE;

    argument = one_argument( argument, arg );
    argument = one_argument( argument, arg2 );

    if ( arg2 == NULL || arg2[0] == '\0' )
	return FALSE;

    if ( obj != get_obj_wear( ch, arg2, ch ) )
	return FALSE;

    if ( fnptr == do_draw )
    {
	int loc = gn_wear_primary;

	if ( get_eq_char( ch, gn_wear_primary ) )
	{
	    if ( get_eq_char( ch, gn_wear_secondary ) )
	    {
		send_to_char("Your hands are full.\n\r", ch );
		return TRUE;
	    }
	    loc = gn_wear_secondary;
	}

	for ( weapon = obj->contains; weapon != NULL; weapon = weapon->next_content )
	{
	    if ( is_name( arg, weapon->name ) )
		break;
	}

	if ( weapon == NULL )
	{
	    act( "$p doesn't seem to contain that weapon.", ch, NULL, obj,
		NULL, NULL, TO_CHAR, SENSE_SIXTH );
	    return TRUE;
	}

	if ( get_obj_weight( weapon ) > (can_carry_w(ch)/15) )
	{
	    send_to_char( "It is too heavy for you to wield.\n\r", ch );
	    return TRUE;
	}

	act( "$n draws $p from $P.", ch, NULL, weapon, obj, NULL,
	    TO_ROOM, SENSE_SIGHT );
	act( "You draw $p from $P.", ch, NULL, weapon, obj, NULL,
	    TO_CHAR, SENSE_SIXTH );

	obj_from_obj( weapon );
	obj_to_char( weapon, ch );
	equip_char( ch, weapon, loc );

	return TRUE;
    }

    if ( fnptr == do_sheath )
    {
	weapon = get_eq_char( ch, gn_wear_primary );

	if ( weapon == NULL || !is_name( arg, weapon->name ) )
	{
	    weapon = get_eq_char( ch, gn_wear_secondary );

	    if ( weapon == NULL || !is_name( arg, weapon->name ) )
	    {
		send_to_char("You don't wield that weapon.\n\r", ch );
		return TRUE;
	    }
	}

	if ( weapon->item_type != ITEM_WEAPON )
	{
	    send_to_char("You can't sheath that.\n\r", ch );
	    return TRUE;
	}

	if ( ( get_obj_weight( weapon ) + get_obj_weight( obj ) )
	    > obj->capacity )
	{
	    send_to_char("There isn't enough room to sheath that.\n\r", ch );
	    return TRUE;
	}

	act( "You sheath $p in $P.", ch, NULL, weapon, obj, NULL,
	    TO_CHAR, SENSE_SIXTH );
	act( "$n sheathes $p in $P.", ch, NULL, weapon, obj, NULL,
	    TO_ROOM, SENSE_SIGHT );

	unequip_char( ch, weapon );
	obj_from_char( weapon );
	obj_to_obj( weapon, obj );

	return TRUE;
    }

*/

    return FALSE;
}


bool spec_flame_blade( CHAR_DATA *ch, char *command, DO_FUN *fnptr,
  char *argument, OBJ_DATA *obj )
{
	/*
    if ( fnptr != (DO_FUN *) violence_update )
	return FALSE;

    if ( obj->wear_loc != gn_wear_primary
    && obj->wear_loc != gn_wear_secondary )
	return FALSE;

    if ( number_bits( 5 ) )
	return FALSE;

    spell_fireball( gsn_fireball, 200, ch, ch->fighting );
*/

    return TRUE;
}

bool spec_glow( CHAR_DATA *ch, char *command, DO_FUN *fnptr, char *argument, OBJ_DATA *obj )
{
    if ( fnptr == NULL )
		return FALSE;

    if ( fnptr == (DO_FUN *) do_kill )
    {
	act( "$n's $o glows brightly as $n attacks!", ch, obj, NULL, TO_ROOM );
	act( "Your $o glows brightly as you attack!", ch, obj, NULL, TO_CHAR );
	return FALSE;
    }

    if ( fnptr == (DO_FUN *) do_get && is_name( argument,  obj->name ) )
    {
    act("$p begins to glow softly as $n reaches for it.", ch, obj, NULL, TO_ROOM );
    act("As you reach for the $o, it begins to glow softly. ", ch, obj, NULL, TO_CHAR );
	return FALSE;
    }
	
	if ( fnptr == (DO_FUN *) do_newput && is_name( argument,  obj->name ) )
    {
    act("$p stops glowing.", ch, obj, NULL, TO_ROOM );
    act("Your $o stops glowing. ", ch, obj, NULL, TO_CHAR );
	return FALSE;
    }

    return FALSE;
}

bool spec_trashbin( CHAR_DATA *ch, char *command, DO_FUN *fnptr, char *argument, OBJ_DATA *obj )
{
    if ( fnptr == NULL )
		return FALSE;

    if ( !str_cmp( command, "toss" ) )
    {
	//act( "$n's $o glows brightly as $n attacks!", ch, obj, NULL, TO_ROOM );
	//act( "Your $o glows brightly as you attack!", ch, obj, NULL, TO_CHAR );
	return FALSE;
    }

    return FALSE;
}


// new stealth system

#if defined(macintosh)
#include <types.h>
#else
#include <sys/types.h>
#endif
#include <stdio.h>
#include <string.h>
#include <time.h>
#include "merc.h"

short armor_penalty( CHAR_DATA *ch )
{
	short penalty = 0;

	OBJ_DATA *armor = get_eq_char( ch, EQUIP_CHEST );
		
		if( armor != NULL )
		{
		penalty = armor_table[armor->value[3]].AP * 2;

		if( !IS_NPC(ch) )
		{
			penalty -= ch->pcdata->learned[gsn_armor_use] / 10; // train it away..
		
			if( penalty < armor_table[armor->value[3]].AP)
			penalty = armor_table[armor->value[3]].AP;
			else
			penalty = armor_table[armor->value[3]].AP;	
		}
		}
	
		// no armr, penalty = 0
		
		return penalty;
}

// ch makes the check against victim
bool perception_check( CHAR_DATA *ch, CHAR_DATA *victim )
{
	short roll1;
	short roll2;
	short roll3;
	
	if( !IS_NPC(ch) )
	roll1= skill_bonus( ch->pcdata->learned[gsn_perception]) / 2;
	else
	roll1 = ch->level * 2;

	if( !IS_NPC(victim) )
	roll2= victim->pcdata->learned[gsn_stalking_and_hiding];
	else
	roll2 = victim->level * 4;

	roll2 += armor_penalty( victim ); //negative

	roll3= open_1d100();
	
	if( !can_see(ch, victim ) )
		return FALSE; // blind
	
	//printf("%s (ch) vs %s (victim) perception check = %d must be greater than %d plus %d\r\n", 
	//ch->name, victim->name, roll1, roll2, roll3 );
	
	if( roll1 > roll2+roll3 )
		return TRUE; // perception has enough to overcome stealth plus randomness
	
	return FALSE; // default - didn't see anything
}


// usage: sneak <direction> incl buildings

void do_sneak( CHAR_DATA *ch, char *argument )
{  
	CHAR_DATA *vch;
    CHAR_DATA *vch_next;
	short direction = -1;
	short rt = 0;
	
	if( argument == NULL )
	{
		send_to_char( "Sneak where, though?\r\n", ch );
		return;
	}
	
	if ( !IS_AFFECTED(ch, AFF_HIDE) )
	{
		send_to_char( "HIDE before SNEAKing.\r\n", ch );
	//REMOVE_BIT(ch->affected_by, AFF_HIDE);
		return;
	}
	
	direction = get_direction( argument );
	
	if( direction == -1 )
	{
	send_to_char( "Sneak where, though?\n\r", ch );
	return;
	}
	
	for ( vch = char_list; vch != NULL; vch = vch_next )
    {
	vch_next	= vch->next;
	if ( vch->in_room == NULL )
	    continue;
	
	if ( vch->in_room == ch->in_room )
	{	
	    if ( vch != ch && perception_check( vch, ch ) ) 
		{
			act( "$N notices you sneaking around.", ch, NULL, vch, TO_CHAR );
			act( "$n just snuck away.", ch, NULL, vch, TO_VICT );	
		}
	    continue;
	}

    }

		//fixme message for successful sneak
	SET_BIT( ch->affected_by, AFF_SNEAK );
	//fixme see if room exists first?
	move_char( ch, direction );
	REMOVE_BIT( ch->affected_by, AFF_SNEAK );
	
	rt = number_range(3,10);
	rt += (get_encumbrance_level(ch) / 10);
	
	if( rt > 10 )
	rt = 10;
	
	add_roundtime( ch, rt );
	show_roundtime( ch, rt );
	
    //if ( IS_AFFECTED(ch, AFF_HIDE) )
	//REMOVE_BIT(ch->affected_by, AFF_HIDE);

    for ( vch = char_list; vch != NULL; vch = vch_next )
    {
	vch_next	= vch->next;
	if ( vch->in_room == NULL )
	    continue;
	
	if ( vch->in_room == ch->in_room )
	{	
	    if ( vch != ch && perception_check( vch, ch ) ) 
		{
			act( "$N notices you sneaking around.", ch, NULL, vch, TO_CHAR );
			act( "$n just snuck in.", ch, NULL, vch, TO_VICT );	
		}
	    continue;
	}

    }
	
    return;
}


void do_hide( CHAR_DATA *ch, char *argument )
{
	CHAR_DATA *vch;
    CHAR_DATA *vch_next;

    if ( IS_AFFECTED(ch, AFF_HIDE) )
	{
		send_to_char( "What are you, paranoid?\r\n", ch );
	//REMOVE_BIT(ch->affected_by, AFF_HIDE);
		return;
	}
	
	if( ch->position != POS_STANDING )
	{
		send_to_char( "You might want to stand up first.\r\n", ch );
		return;
	}	

    for ( vch = char_list; vch != NULL; vch = vch_next )
    {
	vch_next	= vch->next;
	if ( vch->in_room == NULL )
	    continue;
	
	if ( vch->in_room == ch->in_room )
	{	
	    if ( vch != ch && perception_check( vch, ch ) ) 
		{
			act( "$n tries to hide but $N sees $m!", ch, NULL, vch, TO_NOTVICT );
			act( "You try to hide but $N sees you!", ch, NULL, vch, TO_CHAR );
			act( "$n tries to hide but you see $m!", ch, NULL, vch, TO_VICT );	
			return;
		}
	    continue;
	}

    }
	
	//fixme skill check
    if ( !IS_NPC(ch) )
	{
	if( open_1d100( ) < skill_bonus(ch->pcdata->learned[gsn_stalking_and_hiding] ) + armor_penalty(ch) - (get_encumbrance_level(ch) / 10) )
	{
	SET_BIT(ch->affected_by, AFF_HIDE);
    send_to_char( "You skillfully slip into the shadows.\r\n", ch );
	add_roundtime( ch, 3 );
	show_roundtime( ch, 3 );
	return;
	}
	else
	{
    send_to_char( "You fail to hide.\r\n", ch );
	act( "$n looks around for a space to hide before looking disappointed.", ch, NULL, vch, TO_ROOM);
	add_roundtime( ch, 3 );
	show_roundtime( ch, 3 );
	return;
	}	
	}
	else
	{
	SET_BIT(ch->affected_by, AFF_HIDE);
    send_to_char( "You skilfully slip into the shadows.\r\n", ch );
	add_roundtime( ch, 3 );
	show_roundtime( ch, 3 );
	return;
	}

    return;
}

void do_ambush( CHAR_DATA *ch, char *argument )
{
	CHAR_DATA *victim;
	OBJ_DATA *weapon = get_eq_char( ch, EQUIP_RIGHTHAND );
	int boost = 0;
	
	if ( !IS_SET(ch->affected_by, AFF_HIDE ) )
	{
		send_to_char( "You must HIDE first.\n\r", ch );
		return;
	}
	
	if ( ( victim = get_char_room( ch, argument ) ) == NULL )
    {
	send_to_char( "They aren't here.\r\n", ch );
	return;
	}
	
	if (IS_NPC(ch) )
		boost = ch->level * 5;
	else
		boost = ( skill_bonus( ch->pcdata->learned[gsn_ambush] ) + armor_penalty( ch )*2 ) / 2 ;
	
	REMOVE_BIT( ch->affected_by, AFF_HIDE );
	
	act( "$n leaps out of the shadows to attack $N!", ch, NULL, victim, TO_NOTVICT );
	act( "You leap out of the shadows to attack $N!", ch, NULL, victim, TO_CHAR );
	act( "$n leaps out of the shadows to attack you!", ch, NULL, victim, TO_VICT );	
	
	if( boost < 0 )
	{
		printf_to_char( ch, "Your armor is hampering your ability to sneak attack.\r\n");
		boost = 0;
	}
	
	if( !IS_NPC(ch) )
	{
		if( ch->pcdata->learned[gsn_brawling] < 1 )
		{
		if( weapon && weapon->item_type != ITEM_WEAPON )
		{
			send_to_char( "But you know nothing of brawling.  Perhaps get a weapon?\r\n", ch );
			return;
		}
		}
		
		// need a blessed weapon for undead
		if( IS_SET(victim->affected_by, AFF_UNDEAD ) 
	   && (weapon == NULL || 
      ( weapon && !IS_OBJ_STAT(weapon, ITEM_BLESS ) 
	    ) ) )
		{
		send_to_char( "You attack but quickly realize your mundane weapon has no effect!\r\n", ch );
		act( "$n's attack on $N has no effect!", ch, NULL, victim, TO_ROOM );
		
		add_roundtime( ch, 3 );
		show_roundtime( ch, 3 );
		return;
		}
	}
	
	//rt crude, fixme.
	//if( weapon )
	//rt += weapon_table[weapon->value[0]].weapon_speed / 2;

	one_hit( ch, victim, TYPE_UNDEFINED, boost ); 
	
	//add_roundtime( ch, rt );
	//show_roundtime( ch, rt );
}
	

void do_unhide( CHAR_DATA *ch, char *argument )
{
	CHAR_DATA *vch;
    CHAR_DATA *vch_next;
	
	//fixme?

	if ( !IS_AFFECTED(ch, AFF_HIDE) )
	{
	send_to_char( "You aren't hiding.\r\n", ch );
	return;
	}
	
	else
	{
	send_to_char( "You attempt to unhide.\r\n", ch );
	REMOVE_BIT(ch->affected_by, AFF_HIDE);
	
	
	for ( vch = char_list; vch != NULL; vch = vch_next )
    {
	vch_next	= vch->next;
	if ( vch->in_room == NULL )
	    continue;
	
	if ( vch->in_room == ch->in_room )
	{	
	    if ( vch != ch && perception_check( vch, ch ) ) //fixme 10 percent chance should account for perception
		{
			act( "You notice $N looking in your direction as you step out of the shadows.", ch, NULL, vch, TO_CHAR );
			act( "You notice $n step out of the shadows.", ch, NULL, vch, TO_VICT );	
		}
	    continue;
	}

    }
	}
    return;
}

void do_search_room( CHAR_DATA *ch, char *argument )
{
    CHAR_DATA *vch;
    CHAR_DATA *vch_next;
	OBJ_DATA *obj;
	OBJ_DATA *obj_next;
	short rt = number_range(3,10);

    send_to_char( "You search the area carefully, looking for hidden things.\r\n", ch );
	act( "$n starts carefully examining $s surroundings.", ch, NULL, NULL, TO_ROOM );
	
	if( !IS_NPC(ch) )
	

	for ( vch = char_list; vch != NULL; vch = vch_next )
    {
	vch_next	= vch->next;
	if ( vch->in_room == NULL )
	    continue;
	
	if ( vch->in_room == ch->in_room && IS_AFFECTED(vch, AFF_HIDE) )
	{	
	   		if ( vch != ch && perception_check( vch, ch ) ) //fixme 
			{
			REMOVE_BIT(vch->affected_by, AFF_HIDE);
							
							
			act( "$n finds $N hiding in the shadows!", ch, NULL, vch, TO_NOTVICT );
			act( "You find $N hiding in the shadows!", ch, NULL, vch, TO_CHAR );
			act( "$n finds you hiding in the shadows!", ch, NULL, vch, TO_VICT );	
			
			}
			return;
	}
	continue;
	}
	
	 for ( obj = ch->in_room->contents; obj != NULL; obj = obj_next )
	 {
		obj_next = obj->next_content;
		
		if( obj->in_room == NULL )
			continue;
		
		if( obj->in_room == ch->in_room && obj->item_type == ITEM_DOOR && IS_SET(obj->extra_flags, ITEM_HIDDEN) )
		{
			if( !IS_NPC(ch) && skill_bonus( ch->pcdata->learned[gsn_perception] ) + armor_penalty( ch ) > open_1d100() + obj->value[3] )
			{
			REMOVE_BIT( obj->extra_flags, ITEM_HIDDEN );
				
			act( "You discover $p!", ch, obj, NULL, TO_CHAR );
			act( "$n discover $p!", ch, obj, NULL, TO_ROOM );	
			
			}
			
			obj->timer = 2;
		}
		
	 continue;
	 }
	 
	add_roundtime( ch, rt );
    show_roundtime( ch, rt );

}

// ok, not ideal, but whatevs.
bool remove_trap ( OBJ_DATA * obj )
{
	REMOVE_BIT( obj->extra_flags, ITEM_TRAPPED );
	obj->affected = NULL;
	
	return TRUE;
}

short get_trap_diff( OBJ_DATA *obj )
{
	AFFECT_DATA *paf = obj->affected;
	
	while( paf != NULL )
	{
	if( paf->bitvector == AFF_TRAPPED )
	return paf->modifier;
		
	paf = paf->next;
	}

	return -1;
}

//i'm out of values for containers. time to get janky..
void do_disarm( CHAR_DATA *ch, char *argument )
{
    char arg[MAX_INPUT_LENGTH];
    OBJ_DATA *obj;
	int diff = 0;

    one_argument( argument, arg );

    if ( arg[0] == '\0' )
    {
	send_to_char( "Disarm what?\r\n", ch );
	return;
    }

    if ( ( obj = get_obj_here( ch, arg ) ) != NULL )
    {
	/* 'pick object' */
	if ( obj->item_type != ITEM_CONTAINER )
	    { send_to_char( "That's not a container.\r\n", ch ); return; }
	if ( !IS_SET(obj->value[1], CONT_CLOSED) )
	    { send_to_char( "It's not closed.\r\n",        ch ); return; }
	if ( obj->value[2] < 0 )
	    { send_to_char( "It can't be unlocked.\r\n",   ch ); return; }
	if ( !IS_SET(obj->value[1], CONT_LOCKED) )
	    { send_to_char( "It's already unlocked.\r\n",  ch ); return; }
	
	diff = get_trap_diff( obj );
	
	if (diff == -1 )
	{
		printf_to_char( ch, "You find no traps in %s.\r\n", obj->short_descr );
		act( "$n starts to examine $p carefully.", ch, obj, NULL, TO_ROOM );
		
	if (!IS_IMMORTAL(ch))
	{
	add_roundtime( ch, 20 );
	show_roundtime( ch, 20 );
	}
	
	return;
	}
	
	diff += armor_penalty(ch);
	diff -= (get_encumbrance_level(ch) / 10);
	
	//fixme lock + lockpick modifier
    if ( !IS_NPC(ch) && open_1d100( ) > skill_bonus( ch->pcdata->learned[gsn_disarming_traps] ) - diff )
    {
	printf_to_char(ch, "You try to disarm the trap on %s with no success.\r\n", obj->short_descr );
    act( "$n starts to examine $p carefully.", ch, obj, NULL, TO_ROOM );
	
	if (!IS_IMMORTAL(ch))
	{
	add_roundtime( ch, 20 );
	show_roundtime( ch, 20 );
	}
	
	//fixme fumble and boom
	
	return;
    }
	
	printf_to_char(ch, "You hear a plate release as you disarm the trap in %s.\r\n", obj->short_descr );
	act( "You hear something release as $n fiddles with $p.", ch, obj, NULL, TO_ROOM );
	
	remove_trap( obj );
	gain_exp( ch, number_range(25,35) );
	
	if (!IS_IMMORTAL(ch))
	{
	add_roundtime( ch, 20 );
	show_roundtime( ch, 20 );
	}
	
	return;
    }
	else
		printf_to_char(ch, "I see no %s here.\r\n", argument );
	
    return;
}



void do_pick( CHAR_DATA *ch, char *argument )
{
    char arg[MAX_INPUT_LENGTH];
    OBJ_DATA *obj;
	int diff = 0;

    one_argument( argument, arg );

    if ( arg[0] == '\0' )
    {
	send_to_char( "Pick what?\r\n", ch );
	return;
    }

    if ( ( obj = get_obj_here( ch, arg ) ) != NULL )
    {
	/* 'pick object' */
	if ( obj->item_type != ITEM_CONTAINER && obj->item_type != ITEM_DOOR  )
	    { send_to_char( "That's not a container.\r\n", ch ); return; }
	if ( !IS_SET(obj->value[1], CONT_CLOSED) )
	    { send_to_char( "It's not closed.\r\n",        ch ); return; }
	if ( obj->value[2] < 0 )
	    { send_to_char( "It can't be unlocked.\r\n",   ch ); return; }
	if ( !IS_SET(obj->value[1], CONT_LOCKED) )
	    { send_to_char( "It's already unlocked.\r\n",  ch ); return; }
	
	{
	OBJ_DATA *wield = get_eq_char( ch, EQUIP_RIGHTHAND );
    
	if ( wield == NULL || wield->item_type != ITEM_LOCKPICK )
	{ 
		send_to_char( "You need a lockpick in your right hand.\r\n", ch ); 
		return; 
	}

	}
	
	if( IS_SET(obj->extra_flags, ITEM_TRAPPED ) )
	{
	printf_to_char(ch, "You set off the trap in %s and it explodes!\r\n", obj->short_descr );
	act( "You looks surprised as $p suddenly explodes!", ch, obj, NULL, TO_ROOM );
	
	damage( ch, ch, dice(10,10) , TYPE_OBJECT_DAMAGE	);
	
	remove_trap ( obj );
	extract_obj ( obj );
	return;
	}
	
	if ( IS_SET(obj->value[1], CONT_PICKPROOF) )
	    { send_to_char( "You quickly realize that this lock can't be picked.\r\n",             ch ); return; }

	if( obj->value[3] != 0 )
		diff = obj->value[3];
	
	diff += armor_penalty(ch);
	diff -= (get_encumbrance_level(ch) / 10);
	
	//fixme lock + lockpick modifier
    if ( !IS_NPC(ch) && open_1d100( ) > skill_bonus( ch->pcdata->learned[gsn_picking_locks] ) - diff )
    {
	printf_to_char(ch, "You try to pick the lock on %s with no success.\r\n", obj->short_descr );
	act( "$n attempts to pick the lock on $p.", ch, obj, NULL, TO_ROOM );
	
	if (!IS_IMMORTAL(ch))
	{
	add_roundtime( ch, 20 );
	show_roundtime( ch, 20 );
	}
	
	return;
    }
	
	REMOVE_BIT(obj->value[1], CONT_LOCKED);
	printf_to_char(ch, "You hear a *CLICK* as you pick the lock on %s.\r\n", obj->short_descr );
	act( "You hear a *CLICK* as $n picks the lock on $p.", ch, obj, NULL, TO_ROOM );
	gain_exp( ch, number_range(25,35) );
	
	if (!IS_IMMORTAL(ch))
	{
	add_roundtime( ch, 20 );
	show_roundtime( ch, 20 );
	}
	
	if( obj->item_type == ITEM_DOOR )
	{	
		ROOM_INDEX_DATA *to_room = get_room_index( obj->value[0] );
		OBJ_DATA *room_obj;
	
		if ( to_room == NULL )
		return;
	
		for ( room_obj = to_room->contents; room_obj != NULL; room_obj = room_obj->next_content )
		{
			if ( room_obj->item_type == ITEM_DOOR && get_room_index( room_obj->value[0] ) == ch->in_room )
			{
				REMOVE_BIT(room_obj->value[1], CONT_LOCKED);
			}
		}
	}
	
	return;
    }
	else
		printf_to_char(ch, "I see no %s here.\r\n", argument );

    return;
}


//fixme...switch over to coins only or look in containers etc

void do_steal( CHAR_DATA *ch, char *argument )
{
    extern bool is_safe( CHAR_DATA *ch, CHAR_DATA *victim );
    CHAR_DATA *victim;
    int chance;
	int amount;

    if ( argument[0] == '\0'  )
    {
	send_to_char( "Steal from whom?\n\r", ch );
	return;
    }

    if ( ( victim = get_char_room( ch, argument ) ) == NULL )
    {
	send_to_char( "But no one like that is here.\n\r", ch );
	return;
    }

    if ( is_safe( ch, victim ) )
	return;

    if ( victim == ch )
    {
	send_to_char( "You'd be quite the thief if you could steal from yourself.\n\r", ch );
	return;
    }

    //WAIT_STATE( ch, skill_table[gsn_steal].beats );

	chance = open_1d100() + skill_bonus(ch->pcdata->learned[gsn_pickpocketing]) + armor_penalty( ch ) - (get_encumbrance_level(ch) / 10);

    if ( !IS_DEAD(victim) && IS_AWAKE( victim ) && open_1d100() > chance )
    {
	/*
	 * Failure.
	 */
	send_to_char( "You botched the job!\n\r", ch );
	act( "$n just tried to steal from you!\n\r", ch, NULL, victim, TO_VICT   );
	act( "$n reaches into $N's pockets but $N notices!\n\r",  ch, NULL, victim, TO_NOTVICT );
	
	if( IS_NPC(victim) )
		victim->hunting = ch;

	if ( !IS_NPC(ch) && !IS_NPC(victim) )
	{
		//noquit??
	}

	return;
    }

	amount = victim->gold * number_range(1, 10) / 100;
	if ( amount <= 0 )
	{
	    send_to_char( "You couldn't find any silver.\n\r", ch );
		add_roundtime( ch, 1 );
		show_roundtime( ch, 1 );
	    return;
	}

	ch->gold     += amount;
	victim->gold -= amount;
	printf_to_char( ch,  "Success!  You got %d silver coins.\n\r", amount );
	add_roundtime( ch, 1 );
	show_roundtime( ch, 1 );

    return;
}


// Things to do:

// * Should be based on mob speed
// * Possible replace with the rom ported code from mudbytes

#include <stdio.h>
#include <stdlib.h>
#include <time.h>
#ifdef TIME_HUNT
#include <sys/time.h>
#endif
#include "merc.h"

extern const char *dir_name[];
extern const	short	rev_dir;

#define ROOMS_TABLE_SIZE 1063
#define	MAKE_ROOM_HASH(key) (((unsigned int)(key))%ROOMS_TABLE_SIZE)

struct rooms_node {
  ROOM_INDEX_DATA *key;
  int value;
  struct rooms_node *next;
};

struct rooms_table {
  struct rooms_node *buckets[ROOMS_TABLE_SIZE];
};

struct search_node {
  ROOM_INDEX_DATA *room;
  struct search_node *next;
};

struct search_queue {
  struct search_node *head;
  struct search_node *tail;
};

void init_rooms_table (struct rooms_table *rt)
{
  int i;
  for (i = 0; i < ROOMS_TABLE_SIZE; i++)
    rt->buckets[i] = NULL;
}

void destroy_rooms_table (struct rooms_table *rt)
{
  int i;
  struct rooms_node *entry, *temp;

  for (i = 0; i < ROOMS_TABLE_SIZE; i++)
    for (entry = rt->buckets[i]; entry;) {
      temp = entry->next;
      free (entry);
      entry = temp;
    }
}

void rooms_table_add (struct rooms_table *rt, ROOM_INDEX_DATA * key,
  int value)
{
  /* precondition: there is no entry for <key> yet */
  struct rooms_node *temp;
  unsigned int idx;

  idx = MAKE_ROOM_HASH (key);
  temp = (struct rooms_node *) malloc (sizeof (struct rooms_node));
  temp->key = key;
  temp->next = rt->buckets[idx];
  temp->value = value;
  rt->buckets[idx] = temp;
}

int rooms_table_find (struct rooms_table *rt, ROOM_INDEX_DATA * key)
{
  struct rooms_node *entry;
  unsigned int idx;

  idx = MAKE_ROOM_HASH (key);

  entry = rt->buckets[idx];

  while (entry && entry->key != key)
    entry = entry->next;

  return entry ? entry->value : 0;
}

void init_search_queue (struct search_queue *sq)
{
  sq->head = NULL;
  sq->tail = NULL;
}

void destroy_search_queue (struct search_queue *sq)
{
  struct search_node *entry, *temp;

  for (entry = sq->head; entry;) {
    temp = entry->next;
    free (entry);
    entry = temp;
  }
}

ROOM_INDEX_DATA *search_queue_pop (struct search_queue *sq)
{
  ROOM_INDEX_DATA *room;
  struct search_node *entry;

  if (sq->head == NULL)
    return NULL;

  room = sq->head->room;
  entry = sq->head->next;
  free (sq->head);
  sq->head = entry;
  return room;
}

void search_queue_push (struct search_queue *sq, ROOM_INDEX_DATA * room)
{
  struct search_node *entry;

  entry = (struct search_node *) malloc (sizeof (struct search_node));
  entry->room = room;
  if (sq->head == NULL) {
    entry->next = NULL;
    sq->head = entry;
    sq->tail = entry;
  } else {
    entry->next = sq->tail->next;
    sq->tail->next = entry;
    sq->tail = sq->tail->next;
  }
}

struct hunting_data {
  char *name;
  struct char_data **victim;
};

bool exit_ok (EXIT_DATA * pexit, bool go_thru_doors)
{
  if (pexit == NULL || pexit->to_room == NULL)
    return FALSE;
  if (go_thru_doors)
    return TRUE;
  if (IS_SET (pexit->exit_info, EX_CLOSED))
    return FALSE;
  return TRUE;
}

int find_path (ROOM_INDEX_DATA * startp, ROOM_INDEX_DATA * endp,
  CHAR_DATA * ch, int depth, bool in_zone)
{
  struct search_queue s_queue;
  struct rooms_table x_room;

  int i, count = 0, direction = -1;
  bool thru_doors, first_pass = TRUE;
  ROOM_INDEX_DATA *herep, *tmp_room;

#ifdef TIME_HUNT
  char buf[MAX_STRING_LENGTH];
  struct timeval start = { 0, 0 };
  struct timeval stop = { 0, 0 };
  struct timeval result = { 0, 0 };
  double timing;

  gettimeofday (&start, 0);
#endif

  if (depth < 0) {
    thru_doors = TRUE;
    depth = -depth;
  } else {
    thru_doors = FALSE;
  }

  init_rooms_table (&x_room);
  rooms_table_add (&x_room, startp, -1);

  /* initialize queue */
  init_search_queue (&s_queue);
  search_queue_push (&s_queue, startp);

  while ((herep = search_queue_pop (&s_queue)) != NULL) {
    /* only look in the same zone...unless option picked */
    if (herep->area == startp->area || !in_zone) {
      /* for each room look in all directions */
      for (i = 0; i <= 9; i++) {
        /* make sure exit is valid */
        if (exit_ok (herep->exit[i], thru_doors)) {
          tmp_room = herep->exit[i]->to_room;
          if (tmp_room != endp) {
            /* shall we add room to queue?
               count determines total breadth and depth */
            if (!rooms_table_find (&x_room, tmp_room)
              && (count < depth)) {
              count++;
              /* put room on queue for further checking of it's exits */
              search_queue_push (&s_queue, tmp_room);
              /* add room to list of already searched, using it's ancestor for
                 the direction, unless this is the first pass */
              rooms_table_add (&x_room, tmp_room,
                first_pass ? (i + 1) : rooms_table_find (&x_room, herep));
            }
          } else {
            /* we have found our target room */
            direction = rooms_table_find (&x_room, herep);
            /* return direction if first layer */
            if (direction == -1)
              direction = i;
            else
              --direction;
            goto finished;
          }
        }
      }
    }
    first_pass = FALSE;
  }
finished:
  /* couldn't find path */
  destroy_search_queue (&s_queue);
  destroy_rooms_table (&x_room);
#ifdef TIME_HUNT
  gettimeofday (&stop, 0);
  timersub (&stop, &start, &result);
  timing = ((double) result.tv_sec * 1000000) + ((double) result.tv_usec);
  if (IS_IMMORTAL (ch)) {
    sprintf (buf, "Total time was %f microseconds in search of %d rooms.\n",
      timing, count);
    send_to_char (buf, ch);
  }
#endif
  return direction;
}


void hunt_victim (CHAR_DATA * ch)
{
  int dir;
  bool found;
  CHAR_DATA *tmp;

  if (ch == NULL || ch->hunting == NULL || !IS_NPC (ch))
    return;

  /*
   * Make sure the victim still exists.
   */
  for (found = 0, tmp = char_list; tmp && !found; tmp = tmp->next)
    if (ch->hunting == tmp)
      found = 1;

  if (!found || !can_see (ch, ch->hunting)) 
  {
    // act ("$n seethes and looks frustrated.", ch, NULL, ch->hunting, TO_ROOM);
	// we stay silent
	
    ch->hunting = NULL; // turns out, this is necessary
	
	// printf_to_char( ch->hunting, "You feel safe. You have lost %s.\r\n", ch->short_descr);
    return;
  }

  if (ch->in_room == ch->hunting->in_room) {
	  
    multi_hit (ch, ch->hunting, TYPE_UNDEFINED);
	//fixme
    //ch->hunting = NULL;
    return;
  }

  //fixme random vs speed

  ch->wait += number_range(4,7);

  // terminators hunt outside zone.
  if( IS_SET( ch->act, ACT_TERMINATOR ) )
  dir = find_path (ch->in_room, ch->hunting->in_room, ch, -40000, FALSE);
  else
  dir = find_path (ch->in_room, ch->hunting->in_room, ch, -40000, TRUE);

  if (dir < 0 || dir > 9) 
  {
    act ("$n seethes and looks frustrated.", ch, NULL, ch->hunting, TO_ROOM);
    ch->hunting = NULL;
    return;
  }

  /*
   * Give a random direction if the mob misses the die roll.
   */
  if (number_percent () > 75) { /* @ 25% */
    do {
      dir = number_door ();
    }
    while ((ch->in_room->exit[dir] == NULL)
      || (ch->in_room->exit[dir]->to_room == NULL));
  }


  if (IS_SET (ch->in_room->exit[dir]->exit_info, EX_CLOSED)) {
    do_open (ch, (char *) dir_name[dir]);
    return;
  }

  //printf_to_char( ch->hunting, "You feel like you are being hunted.\r\n" ); 
  //add rev dir name or too much? maybe Perception fixme
  move_char (ch, dir);
  return;
}


#if defined(macintosh)
#include <types.h>
#else
#include <sys/types.h>
#endif
#include <stdio.h>
#include <string.h>
#include <time.h>
#include "merc.h"



/*
 * Local functions.
 */
int	hit_gain	args( ( CHAR_DATA *ch ) );
int	mana_gain	args( ( CHAR_DATA *ch ) );
int	move_gain	args( ( CHAR_DATA *ch ) );
void	weather_update	args( ( void ) );
void	char_update	args( ( void ) );
void	obj_update	args( ( void ) );
void	rapid_update	args( ( void ) );



/*
 * Advancement stuff.
 */
void advance_level( CHAR_DATA *ch )
{
	short MTP_add = 25 + 
		( ( get_curr_Judgement( ch ) + 
		get_curr_Intelligence( ch ) + 
		get_curr_Wisdom( ch ) + 
		get_curr_Charm( ch )
		+ ( ( get_curr_Potency(ch) + get_curr_Willpower(ch) ) / 2 ) )
		/ 20);
	short PTP_add = 25 + 
		( (get_curr_Strength( ch ) + 
		get_curr_Endurance( ch ) + 
		get_curr_Dexterity( ch ) + 
		get_curr_Speed( ch )
		+ ( ( get_curr_Potency(ch) + get_curr_Willpower(ch) ) / 2 ) )
		/ 20);
	
	if( IS_NPC(ch) )
	return;
	
	//get PTPs, MTPs and INCREASES
	ch->pcdata->PTPs += PTP_add;
	ch->pcdata->MTPs += MTP_add;
	
	
	// let's increase stats automatically according to race.
/*
	if( ch->level % race_table[ch->race].grow_Strength == 0 )
	{
		printf_to_char( ch, "You feel stronger!\r\n");
		ch->pcdata->perm_Strength++;
	}
	
	if( ch->level % race_table[ch->race].grow_Endurance == 0 )
	{
		printf_to_char( ch, "You feel stronger!\r\n");
		ch->pcdata->perm_Endurance++;
	}
	
	if( ch->level % race_table[ch->race].grow_Dexterity == 0 )
	{
		printf_to_char( ch, "You feel more agile\r\n");
		ch->pcdata->perm_Dexterity++;
	}
	
	if( ch->level % race_table[ch->race].grow_Speed == 0 )
	{
		printf_to_char( ch, "You feel faster!\r\n");
		ch->pcdata->perm_Speed++;
	}

	if( ch->level % race_table[ch->race].grow_Willpower == 0 )
	{
		printf_to_char( ch, "You feel more strong-willed!\r\n");
		ch->pcdata->perm_Willpower++;
	
	if( ch->level % race_table[ch->race].grow_Potency == 0 )
	{
		printf_to_char( ch, "You feel more powerful!\r\n");
		ch->pcdata->perm_Potency++;
	}
	
	if( ch->level % race_table[ch->race].grow_Judgement == 0 )
	{
		printf_to_char( ch, "You feel more judicious!\r\n");
		ch->pcdata->perm_Judgement++;
	}
	
	if( ch->level % race_table[ch->race].grow_Intelligence == 0 )
	{
		printf_to_char( ch, "You feel smarter!\r\n");
		ch->pcdata->perm_Intelligence++;
	}
	
	if( ch->level % race_table[ch->race].grow_Wisdom == 0 )
	{
		printf_to_char( ch, "You feel wiser!\r\n");
		ch->pcdata->perm_Wisdom++;
	}
	
	if( ch->level % race_table[ch->race].grow_Charm == 0 )
	{
		printf_to_char( ch, "You feel more attractive!\r\n");
		ch->pcdata->perm_Charm++;
	}
	*/
	
    return;
}

int get_exp_required( int level )
{
	if( level > 19 )
		return 50000;
	else if( level > 14)
		return 40000;
	else if( level > 9 )
		return 30000;
	else if( level > 4 )
		return 20000;
	else
		return 10000;
}

int get_total_exp_required( int level )
{
	int i;
	int total = 0;

	for( i = 1; i < level; i++ )
		total += get_exp_required( i );

	return total;
}


void gain_exp( CHAR_DATA *ch, int gain )
{
    if ( IS_NPC(ch) || ch->level >= 101 )
	return;

    ch->pcdata->fieldExp += gain;
	
	if( ch->pcdata->fieldExp > get_max_field_exp(ch) )
	{
		ch->pcdata->fieldExp = get_max_field_exp(ch);
		printf_to_char( ch, "Your head pounds with new experiences.  You must rest!\r\n" );
	}
	
	if( ch->pcdata->fieldExp < 0 )
		ch->pcdata->fieldExp = 0;
	
	if( ch->exp < 0)
		ch->exp = 0;
		
    return;
}

/*
 * Regeneration stuff.
 */
int hit_gain( CHAR_DATA *ch )
{
	int gain = (ch->max_hit / 15) + 1;
	
    return UMIN(gain, ch->max_hit - ch->hit);
}



int mana_gain( CHAR_DATA *ch )
{
	int gain = (ch->max_mana / 5) + 3;

    return UMIN(gain, ch->max_mana - ch->mana);
}



int move_gain( CHAR_DATA *ch )
{
    int gain = 1;
	
    return UMIN(gain, ch->max_move - ch->move);
}



void gain_condition( CHAR_DATA *ch, int iCond, int value )
{
    int condition;

    if ( value == 0 || IS_NPC(ch) || ch->level >= LEVEL_HERO )
	return;

    condition				= ch->pcdata->condition[iCond];
    ch->pcdata->condition[iCond]	= URANGE( 0, condition + value, 48 );

    if ( ch->pcdata->condition[iCond] == 0 )
    {
	switch ( iCond )
	{
	case COND_FULL:
	    send_to_char( "You are hungry.\r\n",  ch );
	    break;

	case COND_THIRST:
	    send_to_char( "You are thirsty.\r\n", ch );
	    break;

	case COND_DRUNK:
	    if ( condition != 0 )
		send_to_char( "You are sober.\r\n", ch );
	    break;
	}
    }

    return;
}

int get_current_hour( )
{
	time_t rawtime;
	struct tm * timeinfo;

	time ( &rawtime );
	timeinfo = localtime ( &rawtime );

    return timeinfo->tm_hour;
}

int get_current_minute( )
{
	time_t rawtime;
	struct tm * timeinfo;

	time ( &rawtime );
	timeinfo = localtime ( &rawtime );

    return timeinfo->tm_min;
}

int get_current_month( )
{
	time_t rawtime;
	struct tm * timeinfo;

	time ( &rawtime );
	timeinfo = localtime ( &rawtime );

    return timeinfo->tm_mon;
}

int get_high_temp( )
{
	switch ( get_current_month( ) )
	{
	case 0: return 31;
	case 1: return 36;
	case 2: return 47;
	case 3: return 59;
	case 4: return 71;
	case 5: return 81;
	case 6: return 85;
	case 7: return 82;
	case 8: return 75;
	case 9: return 63;
	case 10: return 48;
	case 11: return 36;
	default: return 0;
	}
}

int get_low_temp()
{
	switch ( get_current_month( ) )
	{
	case 0: return 16;
	case 1: return 21;
	case 2: return 31;
	case 3: return 40;
	case 4: return 51;
	case 5: return 61;
	case 6: return 66;
	case 7: return 65;
	case 8: return 57;
	case 9: return 45;
	case 10: return 34;
	case 11: return 22;
	default: return 0;
	}
}

int get_current_temp()
{
	switch( get_current_hour() )
	{
	case 0:
	return weather_info.tonights_low + 4;
	break;
	case 1:
	return weather_info.tonights_low + 3;
	break;
	case 2:
	return weather_info.tonights_low + 2;
	break;
	case 3:
	return weather_info.tonights_low + 1;
	break;
	case 4:
	return weather_info.tonights_low;
	break;	
	case 5:
	return weather_info.tonights_low + 2;
	break;
	case 6:
	return weather_info.tonights_low + 4;
	break;
	case 7:
	return weather_info.tonights_low + 6;
	break;
	case 8:
	return weather_info.tonights_low + 8;
	break;	
	case 9:
	return weather_info.todays_high - 7;
	break;
	case 10:
	return weather_info.todays_high - 6;
	break;
	case 11:
	return weather_info.todays_high - 5;
	break;
	case 12:
	return weather_info.todays_high - 4;
	break;
	case 13:
	return weather_info.todays_high - 3;
	break;
	case 14:
	return weather_info.todays_high - 2;
	break;
	case 15:
	return weather_info.todays_high - 1;
	break;
	case 16:
	return weather_info.todays_high;
	break;
	case 17:
	return weather_info.todays_high - 1;
	break;
	case 18:
	return weather_info.todays_high - 2;
	break;
	case 19:
	return weather_info.todays_high - 3;
	break;
	case 20:
	return weather_info.todays_high - 5;
	break;
	case 21:
	return weather_info.todays_high - 7;
	break;
	case 22:
	return weather_info.tonights_low + 9;
	break;
	case 23:
	default:
	return weather_info.tonights_low + 7;
	break;
	}
}

/*
 * Update the weather.
 */
void weather_update( void )
{
    char buf[MAX_STRING_LENGTH];
    DESCRIPTOR_DATA *d;
	int minute = get_current_minute();
	int hour = get_current_hour();

    buf[0] = '\0';

	if( minute == 0)
	{
    switch ( hour )
    {
    case  5:
	weather_info.sunlight = SUN_LIGHT;
	strcat( buf, "The day has begun in earnest as the sunlight reaches everywhere.\r\n" );
	break;

    case  6:
	weather_info.sunlight = SUN_RISE;
	strcat( buf, "Dawn streaks towards you as the sun rises in the east.\r\n" );
	break;

    case 12:
        strcat( buf, "The sun has reached its apex above you -- it is noon.\r\n" );
	break;

    case 19:
	weather_info.sunlight = SUN_SET;
	strcat( buf, "The sun dips below the horizon as it disappears into the west.\r\n" );
	break;

    case 20:
	weather_info.sunlight = SUN_DARK;
	strcat( buf, "The night has truly begun as shadows gather into darkness.\r\n" );
	break;

	default:
	break;
    }
	}

    /*
     * Weather change.
     */

	/*
	weather_info.sky = SKY_CLOUDLESS;

	strcat( buf, "The sky is getting cloudy.\r\n" );
	weather_info.sky = SKY_CLOUDY;

	strcat( buf, "It starts to rain.\r\n" );
	weather_info.sky = SKY_RAINING;

	strcat( buf, "The clouds disappear.\r\n" );
	weather_info.sky = SKY_CLOUDLESS;

	strcat( buf, "Lightning flashes in the sky.\r\n" );
	weather_info.sky = SKY_LIGHTNING;

	strcat( buf, "The rain stopped.\r\n" );
	weather_info.sky = SKY_CLOUDY;

	strcat( buf, "The lightning has stopped.\r\n" );
	weather_info.sky = SKY_RAINING;
	*/
	
	if ( hour == weather_info.precip_start_hour && minute == weather_info.precip_start_minute )
	{
	if( get_current_temp( ) < 33 )
	strcat( buf, "It starts to snow.\r\n" );
	else
	strcat( buf, "It starts to rain.\r\n" );
	weather_info.sky = SKY_RAINING;
	}
	
	if ( hour == weather_info.precip_end_hour && minute == weather_info.precip_end_minute && weather_info.sky == SKY_RAINING )
	{
	if( get_current_temp( ) < 33 )
	strcat( buf, "It stops snowing.\r\n" );
	else
	strcat( buf, "It stops raining.\r\n" );
	weather_info.sky = SKY_CLOUDY;
	
	// fixme reset - make code in db.c a func
	
	
	}
	
	// set today's weather
	if( hour == 0 && minute == 0 )
	{
		weather_info.todays_high = get_high_temp( );
		weather_info.tonights_low = get_low_temp( );
		
	weather_info.precip_start_hour = hour + number_range(0,23);
	
	if( weather_info.precip_start_hour > 23 )
		weather_info.precip_start_hour = weather_info.precip_start_hour - 23;	
	
	weather_info.precip_start_minute = minute + number_range(0,59);
	
	if( weather_info.precip_start_minute > 59 )
		weather_info.precip_start_minute = weather_info.precip_start_minute - 59;
	
	weather_info.precip_end_hour = weather_info.precip_start_hour + number_range(0,1);
	
	if( weather_info.precip_end_hour > 23 )
		weather_info.precip_end_hour = weather_info.precip_end_hour - 23;	
	
	weather_info.precip_end_minute = weather_info.precip_start_minute + number_range(0,59);
	
	if( weather_info.precip_end_minute > 59 )
	{
		weather_info.precip_end_hour++;
		weather_info.precip_end_minute = weather_info.precip_end_minute - 59;
	}
	}
	
	// set this hour's weather
	if( minute == 0 )
		weather_info.temp = get_current_temp( );

	
    if ( buf[0] != '\0' )
    {
	for ( d = descriptor_list; d != NULL; d = d->next )
	{
	    if ( d->connected == CON_PLAYING
	    &&   IS_OUTSIDE(d->character)
	    &&   IS_AWAKE(d->character) )
		send_to_char( buf, d->character );
	}
    }

    return;
}

void rapid_update( void )
{
    CHAR_DATA *ch, *ch_next;
	
	for ( ch = char_list; ch != NULL; ch = ch_next )
    {
	ch_next = ch->next;
	
	if ( ch->in_room == NULL )
	continue;
		
	if ( IS_SET( ch->affected_by, AFF_DEAD ) )
	{
		ch->decay -= RAPID_BEAT;
		
		if( ch->decay < 1 )
		{
			if( IS_NPC( ch ) )
			really_kill( ch, FALSE );
			else
			really_kill( ch, TRUE );
		}
    }
	
	if( ch->stun > 0 )
	{
		ch->stun -= RAPID_BEAT;
	
		if( ch->stun < 0 )
		ch->stun = 0;
		else if( IS_NPC(ch) && number_range(0,100) < 10 )
		act( "$n struggles to regain $s bearings!", ch, NULL, NULL, TO_ROOM );
	}
	
	if( ch->wait > 0 )
	{
		ch->wait -= RAPID_BEAT;
	
		if( ch->wait < 0 )
		ch->wait = 0;
	}
	
	if( IS_NPC(ch) && (IS_AFFECTED(ch, AFF_HOLD) || is_affected(ch, gsn_hold_undead)) && number_range(0,100) < 10)
	{
		act( "$n struggles but is held fast!", ch, NULL, NULL, TO_ROOM );
	}
	
	
	if( ch->wait > 0 || ch->stun > 0 || IS_SET( ch->affected_by, AFF_DEAD ) || IS_AFFECTED(ch, AFF_HOLD) || is_affected(ch, gsn_hold_undead) )
	{
		continue;
	}
	
		/*
	if( ch->predelay_info != NULL )
		printf("%s has predelay..arg %s..time %d\r\n", ch->name, ch->predelay_info->argument, ch->predelay_time );
*/

 /* examine delayed functions */
	if ( ch->predelay_time > 0 )
	    ch->predelay_time -= PULSE_PER_SECOND;
	
	if ( ch->predelay_time <= 0 )
	    {
		if ( ch->predelay_info != NULL )
		{
		    (*ch->predelay_info->fnptr) ( ch, ch->predelay_info->argument );

		    free_predelay( ch->predelay_info );
		    ch->predelay_info = NULL;

		    continue;
		}
	    }
	
	if ( !IS_NPC(ch) ) // players don't do this stuff below
		continue;
	
	if( !IS_AWAKE(ch ) ) // let the sleeping sleep
		continue;
		
   if ( ch->desc != NULL ) // switched mobs do nothing
		continue; 

	/* Examine call for special procedure 

	if ( ch->spec_fun != 0 )
	{
	    if ( (*ch->spec_fun) ( ch ) )
		continue;
	}
	
	*/
	
	
	/* Examine call for special procedure */
	if ( ch->spec_fun != 0 )
	{
	    if ( (*ch->spec_fun) ( NULL, NULL, NULL, NULL, ch ) )
		continue;
	}
	
	//fixme randomization...
	if( number_bits(2) == 0 )
		continue;

	// if not standing, stand up...skip a round
	if ( ch->position != POS_STANDING )
	{
		do_stand( ch, "" );
		ch->wait += number_range(3,7);
		continue;
	}
	
	// track ERRRYBODY hunts
	
	if( ch->wait < 1 && ch->hunting != NULL )
	{
		if( number_bits(2) )
		hunt_victim( ch );
	}
	
	if( ch->wait > 1 )
		continue;
	
	// have a prepped spell? then we're in the middle of something, so...
	
	if ( ch->prepared_spell > -1 && spell_table[ch->prepared_spell].spell_fun != spell_null ) 
	{
	
	if ( spell_table[ch->prepared_spell].target == TAR_CHAR_DEFENSIVE || spell_table[ch->prepared_spell].target == TAR_CHAR_SELF 
		|| spell_table[ch->prepared_spell].target == TAR_CHAR_HEALING ) 
	{
		if ( spell_up( ch ) )
			continue;
	}
	else if( spell_table[ch->prepared_spell].target == TAR_CHAR_OFFENSIVE )
	{
		
		if( IS_SET(ch->act, ACT_AGGRESSIVE) && aggro( ch ) )
		continue;
		
		if( IS_SET(ch->act, ACT_KILL_AGGRO) && kill_aggro( ch ) )
		continue;
	}
	}
	
	
	//do nothing
	if( number_range(0, 100) < 20 )
	{
		//ch->wait += number_range(3,7);
		continue;
	} 
	
	/* Scavenge */
	if( IS_SET(ch->act, ACT_SCAVENGER) && scavenge( ch ) )
		continue;

	/* Wander */
	if( !IS_SET(ch->act, ACT_SENTINEL) && wander( ch ) )
		continue;
	
	// aggro
	if( IS_SET(ch->act, ACT_AGGRESSIVE) && aggro( ch ) )
		continue;
		
	if( IS_SET(ch->act, ACT_KILL_AGGRO) && kill_aggro( ch ) )
		continue;
	
	if( IS_SET(ch->act, ACT_SPELLCASTER) && spell_up( ch ) )
		continue;
	}
}

/*
 * Update all chars, including mobs.
 * This function is performance sensitive.
 */
void char_update( void )
{
    CHAR_DATA *ch;
    CHAR_DATA *ch_next;
    CHAR_DATA *ch_save;
    CHAR_DATA *ch_quit;
    time_t save_time;

    save_time	= current_time;
    ch_save	= NULL;
    ch_quit	= NULL;
    for ( ch = char_list; ch != NULL; ch = ch_next )
    {
	AFFECT_DATA *paf;
	AFFECT_DATA *paf_next;

	ch_next = ch->next;


	/*
	 * Find dude with oldest save time.
	 */
	if ( !IS_NPC(ch)
	&& ( ch->desc == NULL || ch->desc->connected == CON_PLAYING )
	&&   ch->save_time < save_time )
	{
	    ch_save	= ch;
	    save_time	= ch->save_time;
	}

	
		if( !IS_NPC(ch) )
		{
		ch->max_move = get_curr_Potency( ch ) / 10;
		
		if( ch->move > ch->max_move )
			ch->move = ch->max_move;
		}
	
	//stop weird "You are now level 1" bug in start room
	if( ch->level > 0 )
	{
	short b;
	short total = 0;
	
	for( b=0; b < MAX_BODY; b++ )
	{
		if( ch->bleed[b] > 0 && !IS_DEAD(ch) )
		{
			if( ch->bandage[b] > 0 )
			{
				//printf_to_char( ch, "The bandages covering your %s degrade.\r\n", body_name[b] );
				//act(  "$n's bandages degrade.", ch, NULL, NULL, TO_ROOM );
				ch->bandage[b]--;
			}
			else
			{
				//printf_to_char( ch, "Your %s bleeds %d health points.\r\n", body_name[b], ch->bleed[b] );
				
				if( ch->hit - ch->bleed[b] < 1 )
				{
				act(  "$n bleeds to death.", ch, NULL, NULL, TO_ROOM );	
				printf_to_char( ch, "Your final drops of blood flow from your %s as you bleed to death.\r\n", body_name[b] );
				}
				//else
				//act(  "Blood drips from $n's wounds.", ch, NULL, NULL, TO_ROOM );
			
				damage( ch, ch, ch->bleed[b], TYPE_BLEED );
				total += ch->bleed[b];
			}
		}
		
	}
			if( total > 0 )
			printf_to_char( ch, "Your wounds bleed a total of %d health points.\r\n", total );
	
	}		
	
		
	    if ( ch->hit  < ch->max_hit && hit_gain(ch) > 0 )
		{
		printf_to_char(ch, "You just recovered %d health.\r\n", hit_gain(ch));
		ch->hit  += hit_gain(ch);
		}
		
		if ( ch->hit > ch->max_hit )
		{
		printf_to_char(ch, "You feel flush.\r\n" );
		ch->hit = ch->max_hit;
		}

	    if ( ch->mana < ch->max_mana && mana_gain(ch) > 0 )
		{
		printf_to_char(ch, "You just recovered %d mana.\r\n", mana_gain(ch));
		ch->mana += mana_gain(ch);
		}
		
		if ( ch->mana > ch->max_mana )
		{
		printf_to_char(ch, "You have a splitting headache.\r\n" );
		ch->mana = ch->max_mana;
		}

	    if ( ch->move < ch->max_move && move_gain(ch) > 0 )
		{
		printf_to_char(ch, "You just recovered %d spirit.\r\n", move_gain(ch));
		ch->move += move_gain(ch);
		}
		
		if ( ch->move > ch->max_move )
		{
		printf_to_char(ch, "You feel sanguine.\r\n" );
		ch->move = ch->max_move;
		}	
		
		//fixme head injuries etc should reduce 
		
		if(!IS_NPC(ch) )
		{
		
		if( ch->pcdata->fieldExp > 1 )
		{
			int tick = 19;
			
			if( ch->in_room->sector_type == SECT_CITY )
				tick = 22;
			
			if ( IS_SET( ch->in_room->room_flags, ROOM_SAFE ) )
				tick = 25;
			
			tick += stat_bonus( get_curr_Judgement( ch ), ch, STAT_JUDGEMENT ) / 5;
			
			if( ch->pcdata->fieldExp < tick )
				tick = ch->pcdata->fieldExp;
			
			ch->pcdata->fieldExp -= tick;
			
			tick *= EXP_BOOST; // fixme should be temporary for testing?
			
			ch->exp += tick;
			
			printf_to_char( ch, "You just gained %d experience.\r\n", tick );
		}
		
		while ( ch->level < 101 && ch->exp >= get_total_exp_required(ch->level+1) )
		{
		printf_to_char( ch, "\r\nYou are now level %d!\r\n\r\n", ch->level+1 );
		ch->level += 1;
		
			// mercy deeds
		if( ch->level == 2 )
		{
		if( IS_SET( ch->act, PLR_COLOR ) )
		send_to_char( "{m", ch );	
			
		printf_to_char( ch, "The gods grant you three favors for free.\r\n");
		ch->pcdata->deeds += 3;
		
		if( IS_SET( ch->act, PLR_COLOR ) )
		send_to_char( "{x", ch );
		}

		//fixme stat gain.
		advance_level( ch );
		}
		
		}


	if ( !IS_NPC(ch) && ch->level < LEVEL_IMMORTAL )
	{
	    if ( ++ch->timer == 15 )
	    {
			
			if( IS_SET( ch->act, PLR_COLOR ) )
			send_to_char( ANSI_BEEP, ch );
		
			if( IS_SET( ch->act, PLR_COLOR ) )
			send_to_char( "{m", ch );
			
		    send_to_char( "YOU HAVE BEEN IDLE FOR A LONG TIME AND WILL SOON BE LOGGED OFF.\r\n", ch );
			
			if( IS_SET( ch->act, PLR_COLOR ) )
			send_to_char( "{x", ch );
			
		    save_char_obj( ch );
	    }

	    if ( ch->timer > 20 )
		ch_quit = ch;

	    gain_condition( ch, COND_DRUNK,  -1 );
	    //gain_condition( ch, COND_FULL,   -1 );
	    //gain_condition( ch, COND_THIRST, -1 );
	}
	

	for ( paf = ch->affected; paf != NULL; paf = paf_next )
	{
	    paf_next	= paf->next;
	    if ( paf->duration > 0 )
		paf->duration--;
	    else if ( paf->duration < 0 )
		;
	    else
	    {
		if ( paf_next == NULL
		||   paf_next->type != paf->type
		||   paf_next->duration > 0 )
		{
		    if ( paf->type > 0 && spell_table[paf->type].msg_off )
		    {		
				printf_to_char( ch, "%s\r\n", spell_table[paf->type].msg_off );
				act(  spell_table[paf->type].msg_off2, ch, NULL, NULL, TO_ROOM );
		    }
		}

		affect_remove( ch, paf );
	    }
	}
	


	/*
	 * Careful with the damages here,
	 *   MUST NOT refer to ch after damage taken,
	 *   as it may be lethal damage (on NPC).
	 */
	 
	 /*
	if ( IS_AFFECTED(ch, AFF_POISON) )
	{
	    act( "$n shivers and suffers.", ch, NULL, NULL, TO_ROOM );
	    send_to_char( "You shiver and suffer.\r\n", ch );
	    damage( ch, ch, 2, gsn_poison );
	}
	*/

    }

    /*
     * Autosave and autoquit.
     * Check that these chars still exist.
     */
    if ( ch_save != NULL || ch_quit != NULL )
    {
	for ( ch = char_list; ch != NULL; ch = ch_next )
	{
	    ch_next = ch->next;
	    if ( ch == ch_save )
		save_char_obj( ch );
	    if ( ch == ch_quit )
		do_quit( ch, "" );
	}
    }

    return;
}



/*
 * Update all objs.
 * This function is performance sensitive.
 */
void obj_update( void )
{
    OBJ_DATA *obj;
    OBJ_DATA *obj_next;

    for ( obj = object_list; obj != NULL; obj = obj_next )
    {
	CHAR_DATA *rch;
	char *message;

	obj_next = obj->next;
	
	//( CHAR_DATA *ch, char *command, DO_FUN *fnptr, char *argument, OBJ_DATA *obj )
	if ( obj->spec_fun != NULL )
	{
	    if ( (*obj->spec_fun) ( NULL, NULL, NULL, NULL, obj ) )
		//if( spec_armsheath(  NULL, NULL, NULL, NULL, obj ) )
		continue;
	}

	if ( obj->timer <= 0 || --obj->timer > 0 )
	    continue;

	switch ( obj->item_type )
	{
	default:              message = "$p vanishes.";         	break;
	case ITEM_DOOR:		  message = NULL; break;
	case ITEM_FOUNTAIN:   message = "$p dries up.";         	break;
	case ITEM_CORPSE_NPC: message = "$p rapidly rots away."; 	break;
	case ITEM_CORPSE_PC:  message = "$p rapidly rots away."; 	break;
	case ITEM_FOOD:       message = "$p decomposes.";			break;
	}

	if(  message != NULL ) // fix for hidden door's usage of timer
	{
		if ( obj->carried_by != NULL )
	    act( message, obj->carried_by, obj, NULL, TO_CHAR );
		else if ( obj->in_room != NULL && ( rch = obj->in_room->people ) != NULL )
	{
	    act( message, rch, obj, NULL, TO_ROOM );
	    act( message, rch, obj, NULL, TO_CHAR );
	}
	}
	
	if ( (obj->item_type == ITEM_CORPSE_NPC || obj->item_type == ITEM_CORPSE_PC )
	&& ( obj->in_room != NULL
	|| ( obj->carried_by != NULL && obj->carried_by->in_room !=NULL ) )&& obj->timer == 0 )
	{
	    OBJ_DATA *o2;
	    ROOM_INDEX_DATA *ir;

	    if ( ( ir = obj->in_room ) == NULL )
		ir = obj->carried_by->in_room;

	    while ( ( o2 = obj->contains ) != NULL )
	    {
			obj_from_obj( o2 );
		    obj_to_room( o2, ir );
	    }

	    extract_obj( obj );
	}
	else if( obj->item_type != ITEM_DOOR ) //anything that's not a door
	    extract_obj( obj );
	else
		SET_BIT( obj->extra_flags, ITEM_HIDDEN );
    }

    return;
}








/*
 * Handle all kinds of updates.
 * Called once per pulse from game loop.
 * Random times to defeat tick-timing clients and players.
 */
void update_handler( void )
{
    static  int     pulse_area;
    static  int     pulse_point;
	static  int     pulse_weather;
	static  int		pulse_rapid_update;

    if ( --pulse_area     <= 0 )
    {
	pulse_area	= PULSE_AREA;
	area_update	( );
    }
	
	if ( --pulse_rapid_update <= 0 )
    {
	pulse_rapid_update = PULSE_RAPID_UPDATE;
	rapid_update ( );
    }

    if ( --pulse_point    <= 0 )
    {
	pulse_point     = PULSE_TICK;
	char_update	( );
	obj_update	( );
    }
	
	if ( --pulse_weather   <= 0 )
    {
	pulse_weather     = PULSE_WEATHER;
	weather_update	( );
    }

    tail_chain( );
    return;
}
  
</pre>
</html>
