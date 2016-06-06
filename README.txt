#Picasso96 SDK Readme

This document should help the programmer of card and chip drivers to get an idea what is necessary to build them, what functionality is needed and in which context the card and chip functions are called.

Graphics boards for the Amiga usually consist of two major components, the Amiga expansion card with the glue logic and the optional video switch and a standard PC-VGA chip. Therefore Picasso96 hardware drivers are usually divided into two parts, a card driver that handles the board specific issues and a chip driver for the VGA display controller. This is primarily to ensure that similar graphics boards like for example the PicassoII(+), the GVP Spectrum and the Piccolo can share the same chip driver and only use different card drivers. This ensures that once the board specific drivers are written, the boards simultaneously get any advances made on the chip side.

But this is not mandatory. If a driver needs to be monolithic or its code will not be used by other similar product, all functionality and code can be put into one single driver module.

Picasso96 board hardware drivers are libraries that currently have only two function calls in addition to the standard library calls:

    BOOL FindCard(register __a0 struct BoardInfo *bi);

and

    BOOL InitCard(register __a0 struct BoardInfo *bi);

FindCard is called in the first stage of the board initialisation and configuration and is used to look if there is a free and unconfigured board of the type the driver is capable of managing. If it finds one, it immediately reserves it for use by Picasso96, usually by clearing the CDB_CONFIGME bit in the flags field of the ConfigDev struct of this expansion card. But this is only a common example, a driver can do whatever it wants to mark this card as used by the driver. This mechanism is intended to ensure that a board is only configured and used by one driver. FindBoard also usually fills some fields of the BoardInfo struct supplied by the caller, the rtg.library, for example the MemoryBase, MemorySize and RegisterBase fields.

InitCard is called in a second stage after the rtg.library allocated other system resources like memory. During this call, all fields of the BoardInfo structure that need to be filled must be initialized. A driver that consists of two separate parts like a card driver and a chip driver calls the init function of the chip driver at that time.

If the board has features that are not the same as the chip driver assumes, like for example, the board uses a different RAM layout than the chip expects or uses other tricks like the changed red and blue lines on the Piccolo, PiccoloSD64 and Spectrum cards, the right time to change the affected parts of the initialisation is after the chip initialisation is completed. Now you can change the RAM configuration or the available RGBFormats without any problems.

The graphics board data and functions are held in one control structure, the BoardInfo structure. It consists of all data that is important for the rtg.library. There is constant data like the memory base or library base pointers, function pointers that point to card or chip specific routines like video switch control or blitter access, and dynamic data used by the rtg.library, like e.g. the current video mode and the active viewport.

If you start a new driver from scratch you should do it in a few steps:

* get the driver to work as a dumb frame buffer,
* get the optional hardware sprite working,
* get the blitter working,
* add special features like flicker fixer, video modules and stuff


**OVERVIEW**

***************

What is needed to get a card to work as a basic dumb frame buffer?

Constant data:

    RegisterBase
    MemoryBase
    MemorySize
    BoardName
    BoardType
    BitsPerCannon
    Flags
    SoftSpriteFlags
    RGBFormats
    MaxHorValue[MAXMODES]
    MaxVerValue[MAXMODES]
    MaxHorResolution[MAXMODES]
    MaxVerResolution[MAXMODES]
    PixelClockArray
    PixelClockCount[MAXMODES]

Functions:

    SetSwitch
    SetColorArray
    SetDAC
    SetGC
    SetPanning
    CalculateBytesPerRow
    CalculateMemory
    GetCompatibleFormats
    SetDisplay
    ResolvePixelClock
    GetPixelClock
    SetClock
    SetMemoryMode
    SetWriteMask
    SetClearMask
    SetReadPlane
    WaitVerticalSync

****************

What is needed to get a card to work with hardware sprite?

    SetSprite
    SetSpritePosition
    SetSpriteImage
    SetSpriteColor

****************

What is needed to get a card to work with blitter acceleration?

Basically you don't need to support all functions immediately as there are default CPU handling routines that can do all the stuff, but if the chip or card offer blitter or some special conversion hardware, there are function pointers that can be overwritten by your driver. Many of the functions also have a xyzDefault routine that points to the CPU function and which can be called if your driver decides not to handle a special case that is not implemented. This way you can focus on those functions that are more important first and implement the other ones as you find time for them.

    BlitPlanar2Chunky
    FillRect
    InvertRect
    BlitRect
    BlitTemplate
    BlitPattern
    DrawLine
    ExchangeRect

****************

Extra stuff

    SetInterrupt
    ResetChip

****************

Special features

    GetFeatureAttrs
    CreateFeature
    SetFeatureAttrs
    DeleteFeature


**IN DETAIL**

Constant data:

```
struct BoardInfo{
    UBYTE         *RegisterBase, *MemoryBase, *MemoryIOBase;
    ULONG        MemorySize;
    char        *BoardName,VBIName[32];
    struct CardBase    *CardBase;
    struct ChipBase    *ChipBase;
    struct ExecBase    *ExecBase;
    struct Library    *UtilBase;
    struct Interrupt    HardInterrupt;
    struct Interrupt    SoftInterrupt;
    struct SignalSemaphore    BoardLock;
    struct MinList    ResolutionsList;
    BTYPE        BoardType;
    PCTYPE        PaletteChipType;
    GCTYPE        GraphicsControllerType;
    UWORD        MoniSwitch;
    UWORD        BitsPerCannon;
    ULONG        Flags;
    UWORD        SoftSpriteFlags;
    UWORD        ChipFlags;        // private, chip specific, not touched by RTG
    ULONG        CardFlags;        // private, card specific, not touched by RTG
    UWORD        BoardNum;
    WORD        RGBFormats;
    UWORD        MaxHorValue[MAXMODES];
    UWORD        MaxVerValue[MAXMODES];
    UWORD        MaxHorResolution[MAXMODES];
    UWORD        MaxVerResolution[MAXMODES];
    ULONG        MaxMemorySize, MaxChunkSize;
    ULONG        *PixelClockArray;
    ULONG        PixelClockCount[MAXMODES];
    APTR __asm    (*AllocCardMem)(register __a0 struct BoardInfo *bi, register __d0 ULONG size, register __d1 BOOL force, register __d2 BOOL system);
    BOOL __asm    (*FreeCardMem)(register __a0 struct BoardInfo *bi, register __a1 APTR membase);
    BOOL __asm    (*SetSwitch)(register __a0 struct BoardInfo *, register __d0 BOOL);
    void __asm    (*SetColorArray)(register __a0 struct BoardInfo *, register __d0 UWORD, register __d1 UWORD);
    void __asm    (*SetDAC)(register __a0 struct BoardInfo *, register __d7 RGBFTYPE);
    void __asm    (*SetGC)(register __a0 struct BoardInfo *, register __a1 struct ModeInfo *, register __d0 BOOL);
    void __asm    (*SetPanning)(register __a0 struct BoardInfo *, register __a1 UBYTE *, register __d0 UWORD, register __d1 WORD, register __d2 WORD, register __d7 RGBFTYPE);
    UWORD __asm    (*CalculateBytesPerRow)(register __a0 struct BoardInfo *, register __d0 UWORD, register __d7 RGBFTYPE);
    APTR __asm    (*CalculateMemory)(register __a0 struct BoardInfo *, register __a1 APTR, register __d7 RGBFTYPE);
    ULONG __asm    (*GetCompatibleFormats)(register __a0 struct BoardInfo *, register __d7 RGBFTYPE);
    BOOL __asm    (*SetDisplay)(register __a0 struct BoardInfo *, register __d0 BOOL);
    LONG __asm    (*ResolvePixelClock)(register __a0 struct BoardInfo *, register __a1 struct ModeInfo *, register __d0 ULONG, register __d7 RGBFTYPE);
    ULONG __asm    (*GetPixelClock)(register __a0 struct BoardInfo *bi, register __a1 struct ModeInfo *mi, register __d0 Index, register __d7 RGBFormat);
    void __asm    (*SetClock)(register __a0 struct BoardInfo *);
    void __asm    (*SetMemoryMode)(register __a0 struct BoardInfo *, register __d7 RGBFTYPE);
    void __asm    (*SetWriteMask)(register __a0 struct BoardInfo *, register __d0 UBYTE);
    void __asm    (*SetClearMask)(register __a0 struct BoardInfo *, register __d0 UBYTE);
    void __asm    (*SetReadPlane)(register __a0 struct BoardInfo *, register __d0 UBYTE);
    void __asm    (*WaitVerticalSync)(register __a0 struct BoardInfo *);
    BOOL __asm    (*SetInterrupt)(register __a0 struct BoardInfo *, register __d0 BOOL);
    void __asm    (*WaitBlitter)(register __a0 struct BoardInfo *);
    void __asm    (*ScrollPlanar)(register __a0 struct BoardInfo *, register __a1 struct RenderInfo *, register __d0 UWORD, register __d1 UWORD, register __d2 UWORD, register __d3 UWORD, register __d4 UWORD, register __d5 UWORD, register __d6 UBYTE);
    void __asm    (*ScrollPlanarDefault)(register __a0 struct BoardInfo *, register __a1 struct RenderInfo *, register __d0 UWORD, register __d1 UWORD, register __d2 UWORD, register __d3 UWORD, register __d4 UWORD, register __d5 UWORD, register __d6 UBYTE);
    void __asm    (*UpdatePlanar)(register __a0 struct BoardInfo *, register __a1 struct BitMap *, register __a2 struct RenderInfo *, register __d0 SHORT, register __d1 SHORT, register __d2 SHORT, register __d3 SHORT, register __d4 UBYTE);
    void __asm    (*UpdatePlanarDefault)(register __a0 struct BoardInfo *, register __a1 struct BitMap *, register __a2 struct RenderInfo *, register __d0 SHORT, register __d1 SHORT, register __d2 SHORT, register __d3 SHORT, register __d4 UBYTE);
    void __asm    (*BlitPlanar2Chunky)(register __a0 struct BoardInfo *, register __a1 struct BitMap *, register __a2 struct RenderInfo *, register __d0 SHORT, register __d1 SHORT, register __d2 SHORT, register __d3 SHORT, register __d4 SHORT, register __d5 SHORT, register __d6 UBYTE, register __d7 UBYTE);
    void __asm    (*BlitPlanar2ChunkyDefault)(register __a0 struct BoardInfo *, register __a1 struct BitMap *, register __a2 struct RenderInfo *, register __d0 SHORT, register __d1 SHORT, register __d2 SHORT, register __d3 SHORT, register __d4 SHORT, register __d5 SHORT, register __d6 UBYTE, register __d7 UBYTE);
    void __asm    (*FillRect)(register __a0 struct BoardInfo *, register __a1 struct RenderInfo *, register __d0 WORD, register __d1 WORD, register __d2 WORD, register __d3 WORD, register __d4 ULONG, register __d5 UBYTE, register __d7 RGBFTYPE);
    void __asm    (*FillRectDefault)(register __a0 struct BoardInfo *, register __a1 struct RenderInfo *, register __d0 WORD, register __d1 WORD, register __d2 WORD, register __d3 WORD, register __d4 ULONG, register __d5 UBYTE, register __d7 RGBFTYPE);
    void __asm    (*InvertRect)(register __a0 struct BoardInfo *, register __a1 struct RenderInfo *, register __d0 WORD, register __d1 WORD, register __d2 WORD, register __d3 WORD, register __d4 UBYTE, register __d7 RGBFTYPE);
    void __asm    (*InvertRectDefault)(register __a0 struct BoardInfo *, register __a1 struct RenderInfo *, register __d0 WORD, register __d1 WORD, register __d2 WORD, register __d3 WORD, register __d4 UBYTE, register __d7 RGBFTYPE);
    void __asm    (*BlitRect)(register __a0 struct BoardInfo *, register __a1 struct RenderInfo *, register __d0 WORD, register __d1 WORD, register __d2 WORD, register __d3 WORD, register __d4 WORD, register __d5 WORD, register __d6 UBYTE, register __d7 RGBFTYPE);
    void __asm    (*BlitRectDefault)(register __a0 struct BoardInfo *, register __a1 struct RenderInfo *, register __d0 WORD, register __d1 WORD, register __d2 WORD, register __d3 WORD, register __d4 WORD, register __d5 WORD, register __d6 UBYTE, register __d7 RGBFTYPE);
    void __asm    (*BlitTemplate)(register __a0 struct BoardInfo *, register __a1 struct RenderInfo *, register __a2 struct Template *, register __d0 WORD, register __d1 WORD, register __d2 WORD, register __d3 WORD, register __d4 UBYTE, register __d7 RGBFTYPE);
    void __asm    (*BlitTemplateDefault)(register __a0 struct BoardInfo *, register __a1 struct RenderInfo *, register __a2 struct Template *, register __d0 WORD, register __d1 WORD, register __d2 WORD, register __d3 WORD, register __d4 UBYTE, register __d7 RGBFTYPE);
    void __asm    (*BlitPattern)(register __a0 struct BoardInfo *, register __a1 struct RenderInfo *, register __d0 WORD, register __d1 WORD, register __d2 WORD, register __d3 WORD, register __d4 WORD, register __d5 WORD, register __d6 UBYTE, register __d7 RGBFTYPE);
    void __asm    (*BlitPatternDefault)(register __a0 struct BoardInfo *, register __a1 struct RenderInfo *, register __d0 WORD, register __d1 WORD, register __d2 WORD, register __d3 WORD, register __d4 WORD, register __d5 WORD, register __d6 UBYTE, register __d7 RGBFTYPE);
    void __asm    (*DrawLine)(register __a0 struct BoardInfo *, register __a1 struct RenderInfo *, register __d0 WORD, register __d1 WORD, register __d2 WORD, register __d3 WORD, register __d4 UBYTE, register __d7 RGBFTYPE);
    void __asm    (*DrawLineDefault)(register __a0 struct BoardInfo *, register __a1 struct RenderInfo *, register __d0 WORD, register __d1 WORD, register __d2 WORD, register __d3 WORD, register __d4 UBYTE, register __d7 RGBFTYPE);
    void __asm    (*ExchangeRect)(register __a0 struct BoardInfo *, register __a1 struct RenderInfo *, register __d0 WORD, register __d1 WORD, register __d2 WORD, register __d3 WORD, register __d4 WORD, register __d5 WORD, register __d6 UBYTE, register __d7 RGBFTYPE);
    void __asm    (*ExchangeRectDefault)(register __a0 struct BoardInfo *, register __a1 struct RenderInfo *, register __d0 WORD, register __d1 WORD, register __d2 WORD, register __d3 WORD, register __d4 WORD, register __d5 WORD, register __d6 UBYTE, register __d7 RGBFTYPE);
    void __asm    (*Reserved0)(register __a0 struct BoardInfo *);
    void __asm    (*Reserved0Default)(register __a0 struct BoardInfo *);
    void __asm    (*Reserved1)(register __a0 struct BoardInfo *);
    void __asm    (*Reserved1Default)(register __a0 struct BoardInfo *);
    void __asm    (*Reserved2)(register __a0 struct BoardInfo *);
    void __asm    (*Reserved2Default)(register __a0 struct BoardInfo *);
    void __asm    (*Reserved3)(register __a0 struct BoardInfo *);
    void __asm    (*Reserved3Default)(register __a0 struct BoardInfo *);
    void __asm    (*Reserved4)(register __a0 struct BoardInfo *);
    void __asm    (*Reserved4Default)(register __a0 struct BoardInfo *);
    void __asm    (*Reserved5)(register __a0 struct BoardInfo *);
    void __asm    (*Reserved5Default)(register __a0 struct BoardInfo *);
    void __asm    (*Reserved6)(register __a0 struct BoardInfo *);
    void __asm    (*Reserved6Default)(register __a0 struct BoardInfo *);
    void __asm    (*Reserved7)(register __a0 struct BoardInfo *);
    void __asm    (*ResetChip)(register __a0 struct BoardInfo *);
    ULONG __asm    (*GetFeatureAttrs)(register __a0 struct BoardInfo *bi, register __a1 APTR FeatureData, register __d0 ULONG Type, register __a2 struct TagItem *Tags);
    struct BitMap * __asm     (*AllocBitMap)(register __a0 struct BoardInfo *, register __d0 ULONG, register __d1 ULONG, register __a1 struct TagItem *);
    BOOL __asm    (*FreeBitMap)(register __a0 struct BoardInfo *, register __a1 struct BitMap *, register __a2 struct TagItem *);
    ULONG __asm    (*GetBitMapAttr)(register __a0 struct BoardInfo *, register __a1 struct BitMap *, register __d0 ULONG);
    BOOL __asm    (*SetSprite)(register __a0 struct BoardInfo *, register __d0 BOOL, register __d7 RGBFTYPE);
    void __asm    (*SetSpritePosition)(register __a0 struct BoardInfo *, register __d0 WORD, register __d1 WORD, register __d7 RGBFTYPE);
    void __asm    (*SetSpriteImage)(register __a0 struct BoardInfo *, register __d7 RGBFTYPE);
    void __asm    (*SetSpriteColor)(register __a0 struct BoardInfo *, register __d0 UBYTE, register __d1 UBYTE, register __d2 UBYTE, register __d3 UBYTE, register __d7 RGBFTYPE);
    APTR __asm    (*CreateFeature)(register __a0 struct BoardInfo *bi, register __d0 ULONG Type, register __a1 struct TagItem *Tags);
    ULONG __asm    (*SetFeatureAttrs)(register __a0 struct BoardInfo *bi, register __a1 APTR FeatureData, register __d0 ULONG Type, register __a2 struct TagItem *Tags);
    BOOL __asm    (*DeleteFeature)(register __a0 struct BoardInfo *bi, register __a1 APTR FeatureData, register __d0 ULONG Type);
    struct MinList    SpecialFeatures;
    struct ModeInfo    *ModeInfo;            /* Chip Settings Stuff */
    RGBFTYPE        RGBFormat;
    WORD        XOffset, YOffset;
    UBYTE        Depth;
    UBYTE        ClearMask;
    BOOL        Border;
    ULONG        Mask;
    struct CLUTEntry    CLUT[256];
    struct ViewPort    *ViewPort;            /* ViewPort Stuff */
    struct BitMap    *VisibleBitMap;
    struct BitMapExtra    *BitMapExtra;
    struct MinList    BitMapList;
    struct MinList    MemList;
    WORD        MouseX, MouseY;        /* Sprite Stuff */
    UBYTE        MouseWidth, MouseHeight;
    UBYTE        MouseXOffset, MouseYOffset;
    UWORD        *MouseImage;
    UBYTE        MousePens[4];
    struct Rectangle    MouseRect;
    UBYTE        *MouseChunky;
    UWORD        *MouseRendered;
    UBYTE        *MouseSaveBuffer;
    ULONG        ChipData[16];        /* for chip driver needs */
    ULONG        CardData[16];        /* for card driver needs */
};
```

RegisterBase:
  This is not referenced from rtg.library directly, it is merely for you to look it up. Normally this is located at the base address of the PC I/O base so that e.g. the timing sequencer index register is located at $3C4 from it, but it can be at any offset you choose, maybe some other offset is more useful to you.

MemoryBase:
  This is the start address of the linear addressable memory of the graphics board. It is needed for direct accesses by the rtg.library.

MemorySize:
  This is the size in bytes of the memory on the board minus the size of reserved memory used e.g. for sprite data like with the Cirrus chips or a buffer for blitting (masks or temp buffers).

BoardName:
  A pointer with the name of your board. You will have to supply this.

BoardType:
  A constant value to identify your board by the rtg.library. You should get a constant assigned by the Picasso96 development team. In the mean time you can use an unused value.

BitsPerCannon:
  The number of significant bits your RAMDAC uses in CLUT based modes. Usually either 6 (64 different colors) or 8 (256 colors) bits per gun.

Flags:
  A field of bits that determine what features your board offers. First, the hardware bits:
    BIB_HARDWARESPRITE
       If set, this type of board has support for a hardware sprite.
       You will have to supply the sprite functions, too.
    BIB_NOMEMORYMODEMIX
       If set, then this board cannot handle access to planar screens
       while displaying chunky ones and vice versa.
    BIB_NEEDSALIGNMENT
       This one is not yet implemented.
    BIB_DBLSCANDBLSPRITEY
       If set, then the y-position of the hardware sprite must be
       doubled when a double scan screen is displayed. This is normally
       the case for hardware sprites that are not controlled by the VGA
       chip but rather by the RAMDAC like on the Merlin board.
    BIB_ILACEHALFSPRITEY
       If set, then the y-position of the hardware sprite must be
       the half of the actual y-position whenever a interlace screen is
       displayed. This is normally the case for hardware sprites that are
       not controlled by the VGA chip but rather by the RAMDAC like on
       the Merlin board.
    BIB_ILACEDBLROWOFFSET
       If set, in interlace modes, the VGA chip needs the offset to the
       first pixel in the next row of the current frame as row offset,
       i.e. during the even frame the odd lines between have to be
       skipped by adjusting in this value. This is so far only true for
       the Tseng ET4000 series.
    BIB_FLICKERFIXER
       This board is equipped with a flicker fixer device that is treated
       as a special feature.
    BIB_VIDEOCAPTURE
       This board is equipped with video capture capability which is also
       treated as a special feature.
    BIB_VIDEOWINDOW
       This board is equipped with video window capability which is also
       treated as a special feature.
    BIB_BLITTER
       If set, this board has a blitter. When available, it can be used
       e.g. to defragment the free memory on the board by moving the
       screens currently on the board.

Now, the user and system modifiable bits:
    BIB_HIRESSPRITE
       If set, mouse has double horizontal resolution, 2*32 bits per line
       like AGA, otherwise 2*16 bits like ECS.
    BIB_BIGSPRITE
       User wants a big sprite image, double each pixel in x and y
       direction.
    BIB_BORDEROVERRIDE
       User wants to override system overscan border prefs.
    BIB_BORDERBLANK
       User wants border blanking, i. e. no overscan border.
    BIB_INDISPLAYCHAIN
       If set, this board switches the Amiga video signal.
    BIB_QUIET
       Not yet implemented, intended to disable startup screens or
       messages.
    BIB_NOMASKBLITS
       If set, the user wants blitting that does not use bit masks.
       This will speed things up with blitters that can not use masks
       like the Cirrus chip.
    BIB_NOC2PBLITS
       The user selected to use CPU for planar to chunky conversions.
       This might be faster on some high-powered Amigas.
    BIB_NOBLITTER
       The user selected to disable all blitter functions.

SoftSpriteFlags:
  If your board has a hardware sprite that cannot be used in some modes, you can set the bit of the RGBFormat(s) of this mode. The rtg.library will the use a software sprite with this mode(s). For example, the CirrusGD542X cannot use the hardware sprite in the TrueColor mode, so the RGBFF_B8G8R8 value is or'ed to this flag field.

RGBFormats:
  All RGB formats that the board can use for screen modes are or'ed together in this field, e.g. the CirrusGD542X uses: RGBFF_PLANAR|RGBFF_CHUNKY|RGBFF_B8G8R8|RGBFF_R5G6B5PC|RGBFF_R5G5B5PC

MaxHorValue and MaxVerValue:
  The maximum horizontal and vertical values that the chip can handle for the CRTC setting in planar, chunky, HiColor, TrueColor and TrueAlpha modes.

MaxHorResolution and MaxVerResolution:
  The maximum horizontal and vertical resolutions that the chip can handle for bitmaps (that might even pan) in planar, chunky, HiColor, TrueColor and TrueAlpha modes.

The difference between those values is:
  MaxHorValue and MaxVerValue stand for the maximum values you can use e.g. for CRTC_HorizontalTotal, i.e. what is displayable. MaxHorResolution and MaxVerResolution on the other hand stand for how big a screen bitmap may be of which maybe only a small part is displayed, e.g. a 2048x2048 bitmap is displayed on a 640x480 screen. So, the "Value"s stand for limits of e.g. CRTC_HorizontalTotal and the "Resolution"s stand for limits of CRTC_RowOffset.

PixelClockArray:
  A pointer to a field of longwords that represent the pixel clock rates (in Hz) the board can handle.

PixelClockCount:
  This indicates how many different clocks are available in planar, chunky, HiColor, TrueColor and TrueAlpha modes, respectively.

**Functions**

Note: If not noted otherwise, registers d0/d1/a0/a1 are scratch.

SetSwitch:
    * a0:    struct BoardInfo
    * d0.w:    BOOL state
  this function should set a board switch to let the Amiga signal pass through when supplied with a 0 in d0 and to show the board signal if a 1 is passed in d0. You should remember the current state of the switch to avoid unneeded switching. If your board has no switch, then simply supply a function that does nothing except a RTS.

SetColorArray:
    * a0:    struct BoardInfo
    * d0.w:    startindex
    * d1.w:    count
  when this function is called, your driver has to fetch "count" color values starting at "startindex" from the CLUT field of the BoardInfo structure and write them to the hardware. The color values are always between 0 and 255 for each component regardless of the number of bits per cannon your board has. So you might have to shift the colors before writing them to the hardware.

SetDAC:
    * a0:     struct BoardInfo
    * d7:    RGBFTYPE RGBFormat
  This function is called whenever the RGB format of the display changes, e.g. from chunky to TrueColor. Usually, all you have to do is to set the RAMDAC of your board accordingly.

SetGC:
    * a0:     struct BoardInfo
    * a1:     struct ModeInfo
    * d0:    BOOL Border
  This function is called whenever another ModeInfo has to be set. This function simply sets up the CRTC and TS registers to generate the timing used for that screen mode. You should not set the DAC, clocks or linear start adress. They will be set when appropriate by their own functions.

SetPanning:
    * a0:     struct BoardInfo
    * a1:     UBYTE *Memory
    * d0:     UWORD Width
    * d1:     WORD XOffset
    * d2:     WORD YOffset
    * d7:    RGBFTYPE RGBFormat
  This function sets the view origin of a display which might also be overscanned. In register a1 you get the start address of the screen bitmap on the Amiga side. You will have to subtract the starting address of the board memory from that value to get the memory start offset within the board. Then you get the offset in pixels of the left upper edge of the visible part of an overscanned display. From these values you will have to calculate the LinearStartingAddress fields of the CRTC registers.

CalculateBytesPerRow:
    * a0:     struct BoardInfo
    * d0:     UWORD Width
    * d7:    RGBFTYPE RGBFormat
  This function calculates the amount of bytes needed for a line of "Width" pixels in the given RGBFormat.

CalculateMemory:
    * a0:     struct BoardInfo
    * a1:     APTR Memory
    * d7:    RGBFTYPE RGBFormat
  This function transforms logical into physical memory addresses. This is normally the same value except for planar bitmaps that use physical offset addresses that are only 1/4 of the logical offset because the VGA chips store these bitmaps in a different way.

GetCompatibleFormats:
    * a0: struct BoardInfo *bi
    * d7: RGBFTYPE RGBFormat
  This function gets a long work representing all RGBFormats that can coexist simultaneously on the board with a bitmap of the supplied RGBFormat. This is used when a bitmap is put to the board in order to check all bitmaps currently on board if they have to leave. This is for example the case of Cirrus based boards whenever a planar bitmap needs to be put to the board, all chunky and Hi/TrueColor bitmaps must leave because the formats cannot share the board due to restrictions by that VGA chips.

SetDisplay:
    * a0:    struct BoardInfo
    * d0:    BOOL state
  This function enables and disables the video display. On VGA chips this is usually done by changing bit 5 of the TS_TSMode register.

ResolvePixelClock:
    * a0:    struct BoardInfo
    * a1:    struct ModeInfo
    * d0.l:    pixelclock
    * d7:    RGBFormat
  This function gets the number of the pixel clock rate that suits the best to the clock rate and RGBFormat supplied and writes the appropiate parameters to the ModeInfo structure.

GetPixelClock:
    * a0:    struct BoardInfo
    * a1:    struct ModeInfo
    * d0.l:    index
    * d7:    RGBFormat
  This function retrieves the effective pixel clock at position "index" of the pixel clock array. On some chips you have to take the given RGBFormat into account, e.g. with the CirrusGD542X you have to divide the raw pixel clock by three to get the effective pixel clock when using TrueColor modes.

SetClock:
    * a0:    struct BoardInfo
  This function writes the clock parameters of the ModeInfo currently active to the hardware.

SetMemoryMode:
    * a0:    struct BoardInfo
    * d7:    RGBFTYPE RGBFormat
  This function sets the memory interface of the chip to use an appropriate setting for the given RGBFormat. For example when using RGBFB_PLANAR, you will have to disable "chain 4 mode".

SetWriteMask:
    * a0:    struct BoardInfo
    * d0:    UBYTE mask
  This function sets the TS_WritePlaneMask register which is used for planar modes.

SetClearMask:
    * a0:    struct BoardInfo
    * d0:    UBYTE mask
  This function sets the GDC_EnableSetReset register to the "mask" value and clears the GDC_SetReset register. This is used for planar modes.

SetReadPlane:
    * a0:    struct BoardInfo
    * d0:    UBYTE Plane
  This function sets the GDC_ReadPlaneSelect register to the "plane" value. This is used for planar modes.

WaitVerticalSync:
    * a0:    struct BoardInfo
  This function waits for the next horizontal retrace.

SetSprite:
    * a0:     struct BoardInfo
    * d0:     BOOL activate
    * d7:    RGBFTYPE RGBFormat
  This function activates or deactivates the hardware sprite.

SetSpritePosition:
    * a0:     struct BoardInfo
    * d7:    RGBFTYPE RGBFormat
  This function sets the hardware mouse sprite position according to the values in the BoardInfo structure.

SetSpriteImage:
    * a0:     struct BoardInfo
    * d7:    RGBFTYPE RGBFormat
  This function gets new sprite image data from the MouseImage field of the BoardInfo structure and writes it to the board.

SetSpriteColor:
    * a0:     struct BoardInfo
    * d0.b:    sprite color index (0-2)
    * d1.b:    red
    * d2.b:    green
    * d3.b:    blue
    * d7:    RGBFTYPE RGBFormat
  This function changes one color of the hardware sprite.

BlitPlanar2Chunky:
  This function is currently only used to transfer planar bitmaps from the Amiga memory to chunky bitmaps on the board.

FillRect:
  This function fills a rectangular area on the board with a given pen or Hi/TrueColor value. It is called by BltBitMap, BltPattern, SetRast and to clear or init a bitmap on the board.

InvertRect:
  This function is used to invert a rectangular area on the board. It is called by BltBitMap, BltPattern and BltTemplate.

BlitRect:
  This function is used to copy a rectangular area on the board. It is called by BltBitMap.

BlitTemplate:
  This function is used to paint a template on the board memory using the blitter. It is called by BltPattern and BltTemplate.

BlitPattern:
  This function is currently not used.

DrawLine:
  This function is currently not used.

ExchangeRect:
  This function is currently not used.

SetInterrupt:
  This function disables or enables the optional board interrupts. Interrupts are disabled automatically during screen switches to avoid problems.

ResetChip:
  This function is currently not used. Its purpose will be to provide a mechanism to initialize the card after a Picasso96 API client has used the board with direct accesses to the hardware.

GetFeatureAttrs:
CreateFeature:
SetFeatureAttrs:
DeleteFeature:
  These functions handle the use of special features like flicker fixer, video and other modules.



