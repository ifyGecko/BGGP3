# BGGP3
Submission for BGGP3: Crash
## Intro
  I am a pretty big retro gaming fan so upon hearing about BGGP3 I immedietly had an idea I knew I would have fun doing,
  crashing doom engines. It seemed like an almost ideal thing to aim for as there are tons of free avilable WAD files to
  use for fuzzing and since the open source release of linuxdoom-1.10 by id software there is an insane amount of open
  source ports to go target.
  
## Doom Background Details
  Doom was released in December of 1993 and is easily stated as one of the most influencial video games of all time.
  It helped define the first person shooter genre of today. Despite being nearly 30 years old it still has an active
  community building new game engines and enhancing already established engines. It still even has people making WAD
  files to this day. A WAD file stands for 'Where's All the Data' and basically holds game data such as map textures
  in what is called lumps. So in essense a WAD is just a bunch of lumps grouped and organized together (cheeky, right).
  
## The Process
  In vein of wanting to take the path of least resistance, the first step I did was apt install all doom engines I could
  find and all free WAD files. With this in hand I started fuzzing each engine using the multipurpose fuzzer 'zzuf'. I
  was quickly shocked at the amount of crashes I was getting and specifically for chocolate-doom and crispy-doom. However,
  my process was being slowed down by SDL pop-up windows indicating errors that I would have to manually close so the
  processes could finally crash.
  
  To overcome this to see the scope of how many crashes I really had on my hands in just a few minutes of fuzzing
  I needed to get around these pop-up windows. With a little research I found the SDL functions that were responsible
  so all I had to do was write a shared library that I could LD_PRELOAD to render these calls inert. The results were
  honestly shocking to me, doing 10K fuzz runs on chocolate-doom and crispy-doom resulted in more crashes than I ever
  would have expected. To make things more managable I just shortened to doing 1000 runs on each and take these as my
  initial for triage.
  
## Triage
  I started with chocolate-doom as it is my favorite of the two and went to github to download the latest source updates.
  After compiling a debug build, I ran zzuf with some of the segfault inducing seeds to confirm the crashes still
  occured. Luckily, out of the managable amount I was planning on testing, they still crashed. 
  
  Now this is where things get really interesting. I was able to find that several of the crashes occcured in the
  zone memory allocator. Since chocolate-doom aims to be a true compatible experience to the original doom release
  in '93 even down to bugs that existed in the engine prior to the release of linuxdoom-1.10 source code it makes
  sense that they use it's dynamic memory allocation code. However, I noted one very odd design choice in the zone
  memory allocator, it defines its size parameter as a signed integer! I immedietly thought that, yeah this could
  cause some problems. However, before I dug deeper I wanted to research more into the, WAD, file format and to
  hopefully find very small WADs or construct my own. This way I was not fuzzing with WADs in the multiple MB size.
  Oddly enough within moments of looking at the WAD file format definition [2] I noticed another odd design choice.
  A WAD file starts with a header that contains magic bytes to specify the type of WAD, the number of lumps (game
  data assets), and a file position to where a directory containing all the info needed to process the lumps.
  Sounds like a decent way of organizing data, except that they defined the 'numlumps' field as a signed integer!
  This gave me a little bit of a laugh and a curious idea, could I just make a WAD file that contains the magic bytes
  and numlumps but set numlumps as -1? Yes, I in fact could and it gloriously segfaults the engine! I now had the
  basis for a small, only 8 bytes, file that could crash a doom engine!
  
  Before I moved on to studying the crash, I wanted to test the file on other doom source ports to see how wide
  spread this could be. I knew that many engines share code from the original linuxdoom-1.10 besides chocolate-doom.
  However, upon researching I found that many modern engines such as Zandronum, GZDoom, and Prboom-plus actually
  do not crash. I do not have concrete reasoning on why but I assume its because they have different dynamic memory
  allocation schemes instead of just copying over the exact code. Despite this, there still are potentially many
  ports that have this problem. Just to point out a couple, there is the Atari Jaguar whose Doom engine source has
  been put into public domain and a GBADoom port on github that shares the same zone memory allocator code base.
  
## The Bug
  So thankfully this bug is easy to track down and rationalize. It all boils down to the zone memory allocator and 
  WAD file format allowing for signed integers where there should only ever be unsigned values.
  
  file contents base64: SVdBRP////8=
  
  sha256sum: 092b95e1dfb2a22d58441239895f7ae2c74db1187e970901e0955ae92f30bb0e
  
  It makes sense that early in the execution the WAD file would need to be initialized into memory since it contains
  the game data. In the function W_AddFile the provided WAD file is parsed and laid into memory. At line 165 in
  w_wad.c in the W_AddFile function the WAD header is read into a struct. However, there is no check to verify that
  a full header read actually occured. This is how you can actually omit the last field of the header to only have
  an 8 byte file. Now that the header has been read, the numlumps field is used to calculate the size of memory
  need for the lumps directory as seen on line 192 in w_wad.c. Because the numlumps field is defined as a negative
  value the next source line end ups invoking a Z_Malloc call requesting a memory block of a negative size.
  
  Into the Z_Malloc, the general idea is that the memory zone has a 'rover' that is used to traverse the linked
  list of memory so a newblock is found by peeking behind the rover's current position to see if it is free. Then
  it begins looping through the linked list checking if the current block is tagged to be free'd or already free
  for an allocation. Now it begins setting up the metadata of the block so the linked list stucture will be
  traversable to include this new block pointing the prev block to it and so on along with indicating extra free
  space in the block. The problem falls on line 329 where the newblock->next->prev dereference now does not
  point to the actual previous field but the id and size field which is not a valid memory address causing the
  segfault.
  
  Due to the limited amount of time I was able to spend on this before submission I was not able to find if there
  was any sort of exploit that could be created out of this bug. I personally don't really think there would be
  but even if not I still found this a pretty neat little crash.
  
## Neat Historical Findings
  While researching this bug I spent a little bit of time digging into historical ties to the zone memory
  allocator and found a few interesting things that I think are note worthy. Besides the historical ties to the
  linuxdoom-1.10 source that I have mentioned such as many source ports or the Atari Jaguar port this bug goes
  a little deeper. Yet also seeming like it would have been noticed and not made it into the original doom source.
  id Software had created many games before Doom with two being Hovertank3D and Wolfenstein3D. Apparently, from
  what I was able to deduce from the internet the memory management in Hovertank3D was used in Wolfenstein3D and
  was the basis to the idea for the zone memory allocator in Doom. Interestingly enough in Hovertank3D they used
  signed int for allocation size [3] but with Wolfenstein3D [4] they chose to use unsigned values!
  
## References
  1) https://github.com/chocolate-doom/chocolate-doom
  2) https://doomwiki.org/wiki/WAD
  3) https://github.com/FlatRockSoft/Hovertank3D/blob/master/MEMMGR.C
  4) https://github.com/id-Software/wolf3d/blob/master/WOLFSRC/ID_MM.C
  
