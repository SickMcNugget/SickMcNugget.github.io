---
layout: post
title: "Terminal Capabilities and ANSI Colour Codes"
date: 2026-04-24 10:21:14 +08:00
categories: linux terminal
published: true
---
The more I use computers, the more I wish to swear off the scourge that is the internet as a whole, even though I use it every day and enjoy the access to limitless information and video/music streaming. Now that I think about it, I just want the good parts without all the bad parts, is that so much to ask for? This thought process is probably just a side-effect of using Linux and having access to manpages, of which I spend a lot of time reading because I enjoy understanding how all the code on my system operates and interoperates. 

I want to look at ANSI escape codes, such as `\e[33m` and `\e[0m`. If anyone's tried to colour the text in their terminal before, these codes should be fairly familiar: make the foreground text yellow and reset the text to default, respectively. `\e` is sometimes written as `^[`, which is a more raw way of representing the escape key, but can cause problems with text editors and formatting, so I would stick to the `readline` way of writing it with `\e`.

My question today is: if I didn't have any internet, how would I work out how to write these sequences in my scripts? This question isn't as easy as you'd expect and it requires some understanding of what a terminal is, how different terminals are configured, and how to read. Let's start at the beginning.

# What the _f&$#_ is a ~kilometre~ terminal?
A terminal is an interface between a user and a computer, allowing for input by the user and output by the computer to occur in a single place. Terminals originally didn't even have a screen connected to them. They instead looked like typewriters, and actually used paper as the interface between user and computer. These terminals were so slow that punch cards were actually the preferred medium for programming for some time. They were serial devices with baud rates around 75, and could handle 10-30 characters per second.

Eventually, Video Display Units were good enough that terminals began to supersede the use of punch cards, but these were still serial devices with physical hardware (more on that later). They had CRT screens, and became the basis of text interaction with an operating system shell that we still use today. It was also around this time that ANSI escape sequences were added to terminals, like the VT100, and became a mainstay of terminal environments. You'll hear about the VT100 family of terminals quite often if you decide to delve into terminals yourself.

These days, dedicated terminals aren't really used anymore. Instead of teletype (tty) devices, pseudo-teletypes (pty) are used, which are software emulated terminals. This is the reason you'll hear many programs refer to terminals as "terminal emulators"; they're software emulated, and the programmer was likely alive when physical terminals were popular, so the distinction mattered at the time. The capabilities (functionality) of these old ttys had to be built into them, whereas modern ptys can have functionality *coded* into them. We now have something called the `termcap database`, which lists which **term**inal **cap**abilities are available in different terminal emulators.

A terminal capability is simply some functionality that the terminal has. We'll cover some of these capabilities below, but to provide some context, it refers to things such as:
- Setting the text foreground/background colour
- Moving the cursor around the screen

Note that termcap is actually a deprecated way of storing terminal capability information, and it now uses `terminfo`-style codes instead. From what I can tell, termcap was limited to two-digit alphanumeric codes for storing capabilities, meaning that they probably were running out of codes, and also the codes weren't particularly readable, nor did they correspond to their functionality particularly well.

Before moving on, I want to touch on the use of the word console vs terminal vs terminal emulator vs virtual console. As far as I can tell, a **terminal** and a **console** are equivalent, and a **virtual console** is equivalent to a **terminal emulator**. In the wild, you're likely to hear that a console is a physical terminal, although a terminal is already a physical terminal, so I'm pretty sure that people are just getting their wires crossed. Either way, I'd say **console** and **terminal emulator** so that everyone knows what you're talking about, although I'm sure there will still be some misunderstandings.

# Configuring a terminal emulator
By default, the `infocmp` command will use the `$TERM` environment variable to look up the capabilities of your terminal emulator using the default terminfo path at `/usr/share/terminfo/`. If the name of another terminfo file is provided to `infocmp`, it will instead look up that entry instead. Let's try it now:
```bash
echo $TERM
# linux

infocmp
#   Reconstructed via infocmp from file: /usr/share/terminfo/l/linux
# linux|Linux console,
#   ...
infocmp konsole
#   Reconstructed via infocmp from file: /usr/share/terminfo/k/konsole
# konsole|KDE console window,
#   ...
infocmp kitty
#   Reconstructed via infocmp from file: /usr/share/terminfo/k/kitty
# kitty|KovId's TTY,
#   ...
infocmp xterm-256color
#   Reconstructed via infocmp from file: /usr/share/terminfo/x/xterm-256color
# xterm-256color|xterm with 256 colors,
#   ...
```
If you're following along, you can see that there are differences between all these files and that the built-in Linux console doesn't have all that many capabilities compared to the other emulators. I'm on KDE Neon whilst writing this and the konsole terminal emulator is actually using the `xterm-256color` capabilities, so I'm not sure what's going on there; likely a compatibility decision.

## Parameterised Strings
I would personally read this from the `terminfo.5` manual page (search for the `Parameterized Strings` section), but I still want to cover it here since it might be somewhat confusing. 

There are three kinds of capabilities:
- boolean capabilities: Either present or not
- Numeric capabilities: A "#" follows the capability name, followed by an integer value
- String capabilities: A "=" follows the capability name, followed by a string of characters making up the capability value. These strings can be verbatim, or allow custom values to be supplied, too. Custom value string capabilities are also known as Parameterised Strings.

Parameterised strings look similar to a printf-style format string. They use a stack under the hood which allows them to perform some complex logic like if-else statements. The parameter processing can change arbitrarily depending on inputs, which we'll see when looking at colours later on.

Let's look at the "set_a_foreground" (setaf) terminfo capability for an example of a parameterised string.
```bash
infocmp linux | grep setaf
```
Outputs
```
setaf=\E[3%p1%dm
```
To go over this format specifier, `\E[3%p1%dm`, we'll split it into it's constituent parts, and explain each one, with reference to the `terminfo.5` manual page (**Parameterized Strings** subheading). The first part is `\E[3`, these are all literal characters, the escape key, left bracket and the literal digit 3. `%p1` means we push the first argument onto the stack, and `%d` prints out the argument as an integer. It ends with a literal 'm' character.

With all this in mind, we could reasonably assume what a valid setaf input looks like. The block below shows what happens when we guess the input values for `%p1`:
```bash
for ((i=0;i<10;i++)); do
  printf "\e[3${i}m"
  printf "Testing what the previous input did\n"
done
```
Assuming you're using the linux terminal emulator, you'll see 8 different text colours printed: black, red, green, yellow (orange), blue, magenta, cyan and white, followed by 2 more repeats of white at the end. This is because those eight colours are portably defined in the ECMA-48 standard (described in the next section), although some terminal emulators offer more colours, which we will get to later.

## An aside about Control Sequences
*Parameterised Strings continues in the next section below*  

ECMA International is an organisation that creates standards for computer systems. One of their standards, [ECMA-48](https://ecma-international.org/wp-content/uploads/ECMA-48_5th_edition_june_1991.pdf), is directly related to the format of our parameterised string above. Any quote blocks below are directly referencing ECMA-48.

### Control Sequence Introducer
> A control sequence is a string of bit combinations starting with the control function **CONTROL SEQUENCE INTRODUCER (CSI)**...
> 
> The format of a control sequence is:  
> CSI P ... P I ... I F  
> where  
> 
> CSI is represented by bit combinations 01/11 (representing ESC) and 05/11 in a 7-bit code...;
> 
> P ... P are Parameter Bytes, which, if present, consist of bit combinations from 03/00 to 03/15;
> 
> I ... I are Intermediate Bytes, which, if present, consist of bit combinations from 02/00 to 02/15.  
Together with the Final Byte F, they identify the control function;  
> 
> F is the Final Byte; it consists of a bit combination from 04/00 to 07/14; it terminates the control sequence and together with the Intermediate Bytes, if present, identifies the control function. Bit combinations 07/00 to 07/14 are available as Final Bytes of control sequences for private (or experimental) use.

Let's break this down a bit. The *ab/cd* sequences above are hexadecimal numbers to represent bit sequences. If we start with the CSI, then we have 1B 5B, which translates to `ESC[` on an [ascii table](https://www.ascii-code.com/). Wonderful, that's exactly how our command started up above. 

Let's move on to Parameter Bytes. **P** can be anything from 30 to 3F, which includes: `0-9:;<=>?`. Again, great, our command uses a '3' as our first parameter byte, and another (arbitrary) integer as our next parameter byte.

Now for Intermediate Bytes. **I** can be anything from 20 to 2F, which includes: `SP!"#$%&'()*+,-./`, where SP is the space character and '-' is a literal hyphen. We don't seem to have that in our string, but the standard is quite insistent on the "if present" wording, so let's assume that our control sequence didn't need an Intermediate Byte.

Let's look at the Final Byte. **F** can be anything from 40 to 7E, which includes: ``@A-Z[\]^_`a-z{|}~``. Great, our string ended with an 'm', meaning we can identify the control function we're calling from the ascii value of 'm' - 6D or 06/13 in ECMA-speak. Thankfully, in the standard, table 3 in section 5.4 provides us the answer:
<table style="text-align: center">
  <thead>
    <tr>
      <th rowspan="2">Row number</th>
      <th colspan="4">Column number</th>
    </tr>
    <tr>
      <th>04</th>
      <th>05</th>
      <th>06</th>
      <th>07</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <td>00</td>
      <td>ICH</td>
      <td>DCH</td>
      <td>HPA</td>
      <td rowspan="16">Private use</td>
    </tr>
    <tr>
      <td>01</td>
      <td>CUU</td>
      <td>SSE</td>
      <td>HPR</td>
    </tr>
    <tr>
      <td>02</td>
      <td>CUD</td>
      <td>CPR</td>
      <td>REP</td>
    </tr>
    <tr>
      <td>03</td>
      <td>CUF</td>
      <td>SU</td>
      <td>DA</td>
    </tr>
    <tr>
      <td>04</td>
      <td>CUB</td>
      <td>SD</td>
      <td>VPA</td>
    </tr>
    <tr>
      <td>05</td>
      <td>CNL</td>
      <td>NP</td>
      <td>VPR</td>
    </tr>
    <tr>
      <td>06</td>
      <td>CPL</td>
      <td>PP</td>
      <td>HVP</td>
    </tr>
    <tr>
      <td>07</td>
      <td>CHA</td>
      <td>CTC</td>
      <td>TBC</td>
    </tr>
    <tr>
      <td>08</td>
      <td>CUP</td>
      <td>ECH</td>
      <td>SM</td>
    </tr>
    <tr>
      <td>09</td>
      <td>CHT</td>
      <td>CVT</td>
      <td>MC</td>
    </tr>
    <tr>
      <td>10</td>
      <td>ED</td>
      <td>CBT</td>
      <td>HPB</td>
    </tr>
    <tr>
      <td>11</td>
      <td>EL</td>
      <td>SRS</td>
      <td>VPB</td>
    </tr>
    <tr>
      <td>12</td>
      <td>IL</td>
      <td>PTX</td>
      <td>RM</td>
    </tr>
    <tr>
      <td>13</td>
      <td>DL</td>
      <td>SDS</td>
      <td style="font-weight: bold">SGR</td>
    </tr>
    <tr>
      <td>14</td>
      <td>EF</td>
      <td>SIMD</td>
      <td>DSR</td>
    </tr>
    <tr>
      <td>15</td>
      <td>EA</td>
      <td>--</td>
      <td>DAQ</td>
    </tr>
  </tbody>
</table>

'm' corresponds the SGR control function. This should be the key to explaining the `\E[3m` sequence from earlier.

### Select Graphic Rendition
Looking up SGR in the standard, we are shown section **8.3.117 SGR - SELECT GRAPHIC RENDITION**.

> Representation: CSI Ps... 06/13  
> Parameter default value: Ps = 0  
> SGR is used to establish one or more graphic rendition aspects for subsequent text. The established aspects remain in effect until the next occurrence of SGR in the data stream, depending on the setting of the GRAPHIC RENDITION COMBINATION MODE (GRCM). Each graphic rendition aspect is specified by a parameter value:  
> 
> 0  default rendition (implementation-defined), cancels the effect of any preceding occurrence of SGR in the data stream regardless of the setting of the GRAPHIC RENDITION COMBINATION MODE (GRCM)  
> 1  bold or increased intensity  
> 2  faint, decreased intensity or second colour  
> 3  italicized  
> 4  singly underlined  
> 5  slowly blinking (less then 150 per minute)  
> 6  rapidly blinking (150 per minute or more)  
> ...  
> 9  crossed-out (characters still legible but marked as to be deleted)  
> ...  
> 21 doubly underlined  
> ...  
> 30 black display  
> 31 red display  
> 32 green display  
> 33 yellow display  
> 34 blue display  
> 35 magenta display  
> 36 cyan display  
> 37 white display  
> ...  
> 39 default display colour (implementation-defined)  
> 40 black background  
> 41 red background  
> 42 green background  
> 43 yellow background  
> 44 blue background  
> 45 magenta background  
> 46 cyan background  
> 47 white background  
> ...  
> 49 default background colour (implementation-defined)  
> ...  
> 51 framed  
> 52 encircled  
> 53 overlined  

Let's start with the "Representation" defined above for SGR. The representation of every SGR command follows the format: `CSI Ps.. 06/13`, which perfectly matches the format of our earlier command, assuming that a `Ps..` of 31 gives us red text, which it does. Note that even though all these different parameters are defined, they need to be supported by the terminal emulator to be accessible by an end user. Thankfully, text foreground colours and bold font are fairly universal, although the range of available colours often differs drastically between different terminal emulators.

Old terminal emulators often only support 8 colours, whereas newer ones support an additional 8 colours, which are similar to the original 8 (think maroon vs red). Modern terminals almost always support 256 colours, and some terminals even have the `$COLORTERM=truecolor` variable set, telling applications that the terminal supports the full range of 16.7 million RGB colours, providing rich colour support.

## Parameterised Strings, continued
Now that we understand parameterised strings somewhat well, we can look at a harder example with conditional logic:
```bash
infocmp xterm-256color | grep setaf
```
Outputs
```
setaf=\E[%?%p1%{8}%<%t3%p1%d%e%p1%{16}%<%t9%p1%{8}%-%d%e38;5;%p1%d%
```
That's probably too hard to read, thankfully `infocmp` has some niceties for formatting entries that look exactly like setaf:

```bash
infocmp xterm-256color -f | grep setaf -A12
```
Now outputs
```
setaf=\E[
        %?
                %p1%{8}%<
                %t3
                %p1%d
        %e
                %p1%{16}%<
                %t9
                %p1%{8}%-%d
        %e38;5;
                %p1%d
        %;
        m,
```
The -f flag shows us conditionals inside these parameterised strings. This one here actually isn't hard to read so we'll go through it quickly. The first block checks if our digit is less than 8. If it is, then we expect the start sequence to be `\E[3<0-7>m`. The second block checks if our digit is less than 16. If it is, then we expect our sequence to be from 9-15, but we subtract 8 before printing out, leaving us with the format `\E[9<0-7>m`. In the final block, which triggers in all other cases, the format is `\E[38;5;<0->m`. Note that in reality, the final block wraps around at 256, i.e. there are only 2^8 different colours available (hence xterm-256color).

Note that when we want to input these sequences manually, we have to take into account the processing that's happening in the terminfo database to print out the correct values. The code block below explains this with comments and examples:
```bash
# prints out the \e[3<0-7>m colours over 8 lines
for ((i=0;i<8;i++)); do
  printf "\e[3${i}mhello\n"
done

# prints out the \e[9<0-7>m colours over 8 lines
# terminfo handles inputs from <8-15> by subtracting 8 from them,
# producing the range \e[9<0-7>m, so we handle that processing ourselves here
for ((i=0;i<8;i++)); do
  printf "\e[9${i}mhello\n"
done

# Prints out the \e[38;5;<0-255>m colours in a 16*16 grid
for ((i=0;i<16;i++)); do
  for ((j=0;j<16;j++)); do
    k=$(((i*16)+j))
    printf "\e[38;5;${k}mtest "
  done
  printf "\n"
done

# Prints out the \e[3<0-7>m, \e[9<0-7>m, and \e[38;5;<16-255> colours
# in a 16*16 grid using the tput command, which will automatically process
# the input integer according to the terminfo parameterisation, as explained
# above.
for ((i=0;i<16;i++)); do
  for ((j=0;j<16;j++)); do
    k=$(((i*16)+j))
    tput setaf $k
    printf "test "
  done
  printf "\n"
done
```

Hopefully you noticed how much slower it is to make a call to 'tput' for every
single word that you want to write, instead of a call to printf by itself.
This is due to the cost of subprocess spawning. 

I'm unsure why the if-else is needed, as the first 16 of the `\e[38;5;` sequences are the
same as the colours present in the `\e[3` and `\e[9` sequences. Likely some kind of
compatibility decision, but I digress.

## A look at capabilities
I thought it would be fun to have a look at the default capabilities in the Linux terminal emulator. Here's a listing of some important ones:
- am (auto-margin): automatically inserts newline at the end of a line (e.g. line wrap).
- smam/rmam: enable/disable auto-margin.
- bce: erase the screen with the current background colour. For example, vim with a colourscheme needs this, assuming the colourscheme background is different to the system default
- ccc: terminal can re-define existing colors
- mir: safe to move while in insert mode
- tbc: Clear all tab stops
- hts: set a tab in every row, current columns
- it: tabs every # spaces
- colors: maximum number of colors on screen
- pairs: maximum number of colour pairs on screen
- home: moves the cursor to the upper left corner of the screen
- clear: clear screen and home cursor
- ri: turn on reverse video mode. This inverts the colour of the background and the foreground of the text. So green text on a black background becomes black text on a green background.
- setab: Set background color to #1, using ANSI escape
- setaf: Set foreground color to #1, using ANSI escape

## The Linux Virtual Console
The Linux virtual console is implemented primarily in the [vt.c file](https://github.com/torvalds/linux/blob/master/drivers/tty/vt/vt.c) in the Linux source tree. Another struct, called `vc_data` in [console_struct.h](https://github.com/torvalds/linux/blob/master/include/linux/console_struct.h#L120) contains an integer flag `vc_decawm` which sets autowrap mode (a.k.a am in terminfo).

Back in `vt.c`, we can see some familiar functionality in the driver code, such as generating rgb colours in `rgb_from_256`, resetting the terminal to normal settings in `reset_terminal`, and handling the SGR control function in `csi_m` (including the enum above the function with all the corresponding control codes).

The code block below shows the three places that auto-wrapping is used. The first two show how the value can be set, and the final function shows how the value is used to adjust the wrapping behaviour in write mode.
```c
/* we can trigger this on the console via "\e[7h" (on) or "\e[7l" (off).
 * In terminfo this is mapped to *smam* and *rmam* */
static void csi_DEC_hl(struct vc_data *vc, bool on_off)
{
    unsigned int i;

    for (i = 0; i <= vc->vc_npar; i++)
        switch (vc->vc_par[i]) {
            case CSI_DEC_hl_AUTOWRAP:
                vc->vc_decawm = on_off;
                break;
            ...
        }
}

/* This function is called when the console is initialised, inside of
 * vc_init, and also when the console receives "\ec" */
static void reset_terminal(struct vc_data *vc, int do_clear)
{
	unsigned int i;
    ...
	vc->vc_decawm		= 1;
    ...
}


/* This function handles user-input and displays it back to the user.
 * vc_decawm in this case prevents wrapping from happening when it should */
static int vc_con_write_normal(struct vc_data *vc, int tc, int c,
                               struct vc_draw_region *draw)
{
    ...

    if (vc->state.x == vc->vc_cols - 1) {
        vc->vc_need_wrap = vc->vc_decawm;
        ...
    }
    ...
}
```

The above code makes no reference to termcap or terminfo, but follows the conventions put forth for valid keybinds. As a result, the terminfo database really has no bearing on what the terminal emulator actually supports/does, unless it is maintained and updated as new features are added and capabilities are changed. For example, The key combination `\ec` causes the terminal to clear for the virtual console, but this is only mentioned underneath the `rs1` entry in infocmp. 

Also, note that `\ec` is defined in ECMA-48 below:
> RIS - RESET TO INITIAL STATE  
Notation: (Fs)  
Representation: ESC 06/03  
> 
> RIS causes a device to be reset to its initial state, i.e. the state it has after it is made operational. This may imply, if applicable: clear tabulation stops, remove qualified areas, reset graphic rendition, put all character positions into the erased state, move the active presentation position to the first position of the first line in the presentation component, move the active data position to the first character position of the first line in the data component, set the modes into the reset state, etc.

Let's look into `rs1` to see what's happening:
```bash
infocmp linux -1 | grep rs1
```
Outputs
```
rs1=\Ec\E]R
```

We know that `\Ec` is RIS. So what's `\E]R`?

It turns out that `\E]R` is undefined, as in it's operating system dependent. Only the `\E]` sequence is defined in ECMA-48, the R is chosen somewhere in the driver source code within the Linux source. Below is the relevant ECMA-48 entry and the corresponding code in the Linux source that decides the functionality:

> OSC - OPERATING SYSTEM COMMAND  
Notation: (C1)  
Representation: 09/13 or ESC 05/13  
> 
> OSC is used as the opening delimiter of a control string for operating system use. The command string following may consist of a sequence of bit combinations in the range 00/08 to 00/13 and 02/00 to 07/14. The control string is closed by the terminating delimiter STRING TERMINATOR (ST). The interpretation of the command string depends on the relevant operating system. 

```c
/*
 * Handle a character (@c) following an ESC (when @vc is in the ESesc state).
 * E.g. previous ESC with @c == '[' here yields the ESsquare state (that is:
 * CSI).
 */
static void handle_esc(struct tty_struct *tty, struct vc_data *vc, u8 c)
{
    vc->vc_state = ESnormal;
    switch (c) {
        ...
        case ']':
            vc->vc_state = ESnonstd;
            break;
        ...
    }
}

/* Here they define 'P', 'R' and '0'-'9' to have custom OS-based functionality.
 * I'm just going to look at reset_palette for this article. */
static void do_con_trol(struct tty_struct *tty, struct vc_data *vc, u8 c)
{
    ...
    switch(vc->vc_state) {
        ...
        case ESnonstd:	/* ESC ] aka OSC */
            switch (c) {
                case 'P': /* palette escape sequence */
                    vc_reset_params(vc);
                    vc->vc_state = ESpalette;
                    return;
                case 'R': /* reset palette */
                    reset_palette(vc);
                    break;
                case '0' ... '9':
                    vc->vc_state = ESosc;
                    return;
            }
            vc->vc_state = ESnormal;
            return;
        ...
    }
    ...
}

/* Pretty simple set-to-default function */
void reset_palette(struct vc_data *vc)
{
	int j, k;
	for (j=k=0; j<16; j++) {
		vc->vc_palette[k++] = default_red[j];
		vc->vc_palette[k++] = default_grn[j];
		vc->vc_palette[k++] = default_blu[j];
	}
	set_palette(vc);
}
```

That's enough of that. It's pretty clear at this point that we're looking at decades of decisions that have led to the architecture of terminals, virtual or otherwise, as they are today. It all makes a lot of sense when you zoom out like this, but trying to make sense of it at a high level can lead to a great deal of confusion when you want to *understand* why certain codes do certain things. Time to wrap up.

# To conclude
To reiterate my original question, if I didn't have any internet, how would I work out how to write these sequences in my scripts? I think I have an answer now, although it isn't that straight forward. First of all, `infocmp` is your best friend, and the terminfo database at `/usr/share/terminfo/` is a good friend. If you have the `tput` command available, which most systems do, then you can easily toggle terminal capabilities using this command. Otherwise, you'll need to read the parameterised strings in the terminfo database with help from the `terminfo.5` manual page. Otherwise, there isn't much more to it. I wouldn't call any of this tribal knowledge, since I was able to research it online, but it's not immediately apparent how terminal emulators work.
