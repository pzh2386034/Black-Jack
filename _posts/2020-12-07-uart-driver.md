``` c
static const fle_operations tty_fops = {
    .read = tty_read,
    .write = tty_write,
    .open = tty_open,
};/* 用户空间 */
static const struct tty_operations uart_ops = {
    .open = uart_open,
    .close = uart_close,
    .write = uart_write,
};/* line discipline */
struct tty_ldisc_ops tty_ldisc_N_TTY = {
    .open = n_tty_open,
    .read = n_tty_read,
    .write = n_tty_write,
    .receie_buf = n_tty_receive_buf,
}; /* serial_driver */
static struct uart_ops s3c24xx_serial_ops = {
    .start_tx = s3c24xx_serial_start_tx,
    .startup = s3c24xx_serial_startup,
};/*harware*/
```

`dirvers/tty/serial/samsung.c`

``` c
static struct uart_driver s3c24xx_uart_drv = {
	.owner		= THIS_MODULE,
	.driver_name	= "s3c2410_serial",
	.nr		= CONFIG_SERIAL_SAMSUNG_UARTS,
	.cons		= S3C24XX_SERIAL_CONSOLE,
	.dev_name	= S3C24XX_SERIAL_NAME,
	.major		= S3C24XX_SERIAL_MAJOR,
	.minor		= S3C24XX_SERIAL_MINOR,
};
```

