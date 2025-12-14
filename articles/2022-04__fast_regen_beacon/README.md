# Fast Regeneration Beacon Timing Analysis

The designer of the fast regeneration beacon is Zomie101. TMA Discord message link: https://discord.com/channels/594920796867133446/594937596786900993/894035094246575214

1. Uses a 32gt clock to block the beacon. The beacon refreshes potion effects every 80gt. The LCM of these is 160, effectively doubling the beacon's working cycle—meaning the player's Regeneration I effect only refreshes every 160gt;

2. Regeneration I heals by "restoring one health point whenever the potion effect's remaining time is a multiple of 50gt." A full beacon gives players 340gt of Regeneration. Combined with the 80gt refresh cycle, we can see that players only heal once when Regeneration I has 300gt remaining—actually recovering 1 health per 80gt, which is worse than normal healing of 1 health per 50gt;

3. After extending the Regeneration I effect refresh cycle to 160gt, players can heal when Regeneration I has 300gt, 250gt, and 200gt remaining—three times total. This means 3 health per 160gt, averaging 1.5 health per 80gt, 1.5 times the healing speed of a regular beacon.

## File Contents

The entire Jupyter Notebook contains code I used to draw timing diagrams with Matplotlib
