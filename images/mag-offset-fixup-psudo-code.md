```
float m5_mag_offset[3];

// AK8963 range +- 4900uT
#define MAG_LIM 49000

float mx_raw_min = MAG_LIM, mx_raw_max = -MAG_LIM;
float my_raw_min = MAG_LIM, my_raw_max = -MAG_LIM;
float mz_raw_min = MAG_LIM, mz_raw_max = -MAG_LIM;

// Guess compass offset values

int mag_calib_state = 0;
// CONFIG_MAG_H_INTENSITY 296 "Geomagnetic horizontal intensity"
// CONFIG_MAG_V_INTENSITY 372 "Geomagnetic vertical intensity"
#define MAG_HOR CONFIG_MAG_H_INTENSITY
#define MAG_VER CONFIG_MAG_V_INTENSITY

bool guess_mag_offset(void)
{
    bool changed = false;
    if (mag_calib_state == 0
        && mx_raw_max - mx_raw_min > 2 * MAG_HOR * 0.8
        && my_raw_max - my_raw_min > 2 * MAG_HOR * 0.8) {
        m5_mag_offset[0] = (mx_raw_min + mx_raw_max)/2;
        m5_mag_offset[1] = (my_raw_min + my_raw_max)/2;
        mag_calib_state = 1;
        changed = true;
    }
    if (mag_calib_state == 1
        && mz_raw_max - mz_raw_min > 2 * MAG_VER * 0.8) {
        m5_mag_offset[2] = (mz_raw_min + mz_raw_max)/2;
        mag_calib_state = 2;
        changed = true;
    }
    return changed;
}

void loop()
{
    // Read the mag data into mx, my, mz
    code_to_read_mxyz
 
    if (mx < mx_raw_min)
        mx_raw_min = mx;
    if (mx > mx_raw_max)
        mx_raw_max = mx;
    if (my < my_raw_min)
        my_raw_min = my;
    if (my > my_raw_max)
        my_raw_max = my;
    if (mz < mz_raw_min)
        mz_raw_min = mz;
    if (mz > mz_raw_max)
        mz_raw_max = mz;

    guess_mag_offset();

    if (mag_calib_state == 0) {
        M5.Lcd.printf("Rotate horizontally");
    } else if (mag_calib_state == 1) {
        M5.Lcd.printf("Turn upside down");
    } else {
        M5.Lcd.printf("Completed!");
    }

    //
}
```
