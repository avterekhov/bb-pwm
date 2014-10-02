bb-pwm
======

Kernel patch for beaglebone PWM period control. Tested on 3.8-rt kernel.

After applying this patch to the kernel you can independently control periods for both channels of the same chip.

Problem
-------

*You can't change PWM period on beaglebone (and beaglebone black) if you use two PWM channels corresponding to the same PWM chip.*

For example, if you want to use EHRPWMB and EHRPWMA at the same time, you have to use the default values of the PWM period (500000 ns or 2 kHz on 3.8 kernel):

        echo am33xx_pwm > /sys/devices/bone_capemgr.*/slots
        echo bone_pwm_P8_13 > /sys/devices/bone_capemgr.*/slots
        echo bone_pwm_P8_19 > /sys/devices/bone_capemgr.*/slots

Now

        cat /sys/devices/ocp.*/pwm_test_P8_13.*/period

returns

        500000

But if you try to change the period

        echo 50000 > /sys/devices/ocp.*/pwm_test_P8_13.*/period

you will recieve an error:

        -sh: echo: write error: Invalid argument
 

You can still change the period if you use only one of the channels

        echo am33xx_pwm > /sys/devices/bone_capemgr.*/slots
        echo bone_pwm_P8_13 > /sys/devices/bone_capemgr.*/slots

so

        echo 50000 > /sys/devices/ocp.*/pwm_test_P8_13.*/period

passes without an error and

        cat /sys/devices/ocp.*/pwm_test_P8_13.*/period

correctly returns

        50000

But now if you try to export the other PWM channel

        echo bone_pwm_P8_19 > /sys/devices/bone_capemgr.*/slots

it will fail to do so:

        ls /sys/devices/ocp.*/pwm_test_P8_19.*/

returns

        modalias  power  subsystem  uevent

i.e. you will have no period, duty and polarity files to control this PWM channel.

The reason for this behavior is that the Texas Instruments [AM335x PWM Driver](http://processors.wiki.ti.com/index.php/AM335x_PWM_Driver's_Guide) has two PWM channels for the same PWM chip and they share the same frequency / period controller. However, the current logic of the kernel code, as far as I understand it, does not assume that different PWM channels can be interdependent. So, the TI EHR PWM driver simply forbids you to change the value of the period if there are more than one channel active for the same chip.

Solution
--------

As far as I know, there is no solution that would not require rebuilding the kernel. There is one solution suggested [here](http://saadahmad.ca/using-pwm-on-the-beaglebone-black/). It will not allow you to change PWM period when you have both channels exported, but it will allow you to export one device, change period and export the other device then.

The following solution will allow you to change the period at any time. It involves hacking with PWM channel code which I summarize in this [patch](http://raw.githubusercontent.com/avterekhov/bb-pwm/master/ti_ehr_pwm.patch). Note that the patch was made for 3.8-rt kernel. For those who have no experience recompiling linux kernel and installing it on beaglebone I can recommend [these instructions](http://dev.ardupilot.com/wiki/building-for-beaglebone-black-on-linux/). If you follow them, you need to apply this patch just before building the kernel, i.e. after you execute

        cd kernel

for the second time, you copy the patch to the project's root folder:

        cp <somewhere>/ti_ehr_pwm.patch ..

apply the patch

        patch -p1 < ../ti_ehr_pwm.patch

and only then you build the kernel.

 

With this kernel you must be able to change PWM period without any problems. To test this, try:

        echo am33xx_pwm > /sys/devices/bone_capemgr.*/slots
        echo bone_pwm_P8_13 > /sys/devices/bone_capemgr.*/slots
        echo bone_pwm_P8_19 > /sys/devices/bone_capemgr.*/slots
        echo 50000 > /sys/devices/ocp.*/pwm_test_P8_13.*/period[/code]

The PWM should be set correctly.

        cat /sys/devices/ocp.*/pwm_test_P8_13.*/period

returns

	50000

And now it should be the same for both channels:

        cat /sys/devices/ocp.*/pwm_test_P8_19.*/period

returns

        50000
 
Note that changing the period will fail if the target period value is smaller than the duty cylce for any of the channels.

Check http://randomlinuxhacks.wordpress.com