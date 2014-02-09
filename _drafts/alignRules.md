---

layout: post


title: File Names Alignment Rules


date:   2014-02-09 10:49:01
---

Normally the <acronym title="Hierarchical File System">HFS+</acronym> disks are formatted ignoring the file name case, so if we have winter.jpg and WINTER.JPG they cannot be created on same directory ffff.

***

When the HFS+ partitions are formatted with case sensitive support the file names winter.jpg and WINTER.JPG are two different file names and can be created on same directory.

It is possible to let VisualDiffer determine the file name alignment case algorithm looking at HFS partition case, so three scenarios are possible



If the last scenario is true, the alignment will try to be smart, first it searches if a match case is available (winter.jpg with winter.jpg) then it tries to align with the most similar name.

# Align by User Defined Rules 

There are scenarios where it is necessary to align files having different names.

The most simple scenario has files with same name but different extension as shown below