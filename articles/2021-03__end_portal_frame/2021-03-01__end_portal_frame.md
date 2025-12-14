# The Quirky End Portal Frame: It Also Works Vertically!

Revisited the End Portal frame structure detection, organized the history of this feature, and posting again.

Mojang: Nobody understands structure detection better than me.



To allow Snow Golems and Iron Golems to be summoned with any rotated structure, a structure is rotated and compared multiple times during detection. The End Portal frame detection logic works the same way.

This detection logic only appeared in 1.9, and has remained basically unchanged up to the current 1.16.4, except that in 1.9, the 3×3 square area enclosed by the End Portal frames (both horizontal and vertical placements; see Image 2 for vertical case) must not have blocks obstructing it. The wiki also notes that before 1.10, End Portal activation could be blocked by blocks (Image 4). In 1.8 and earlier (only tested 1.7 and 1.8), only horizontally placed End Portal frame structures could properly open the portal, and there was no corner activation feature.

Note: The cubic region marked by yellow stained glass indicates positions where the End Portal structure can be successfully activated (Image 1). You only need to have an End Portal frame filled with an Eye of Ender within this range to activate the End Portal. This means normally horizontal End Portal frames can also be reactivated by placing an empty frame at a corner of the structure and filling it with an Eye of Ender (Image 3).

~~I left these quirky portals at OSTC Lab on CCS, feel free to check them out if you're interested~~. (Why strikethrough: because the server shut down)

Mojang, what on earth are you doing, Mojang

("麻将" is a Chinese homophonic joke for Mojang)


![image1](./img/image1.png)

![image2](./img/image2.png)

![image3](./img/image3.png)

![image4](./img/image4.png)

![image5](./img/image5.png)
