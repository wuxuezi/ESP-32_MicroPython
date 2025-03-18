import time
import random
from machine import SPI, Pin, ADC
from fonts.ufont import BMFont
from st7789 import ST77XX

# 初始化 SPI 和显示屏
spi = SPI(1, 40000000, sck=Pin(14), mosi=Pin(13))
display = ST77XX(spi=spi, cs=6, dc=4, rst=17, bl=10, width=240, height=320, rotate=1, offset=(0, 0, 240, 320))

# 载入中文字体
font = BMFont("unifont-14-12917-16.v3.bmf")

# 摇杆初始化
JOYSTICK_X = ADC(Pin(1, Pin.IN))  # X 轴接 GPIO1
JOYSTICK_Y = ADC(Pin(2, Pin.IN))  # Y 轴接 GPIO2
JOYSTICK_VCC = Pin(7, Pin.OUT)  # VCC 接 GPIO7
JOYSTICK_VCC.value(1)  # 摇杆供电

# 配置ADC的电压范围（ESP32 ADC默认范围为0-4095）
JOYSTICK_X.atten(ADC.ATTN_11DB)  # 设置输入电压范围为0-3.3V
JOYSTICK_Y.atten(ADC.ATTN_11DB)

# 配置 ADC 分辨率为 12 位（0-4095）
JOYSTICK_X.width(ADC.WIDTH_12BIT)
JOYSTICK_Y.width(ADC.WIDTH_12BIT)

# 游戏区域大小
GRID_WIDTH = 15  # 横向块数
GRID_HEIGHT = 20  # 纵向块数
BLOCK_SIZE = min(display.width // GRID_WIDTH, display.height // GRID_HEIGHT)  # 块大小（自动计算）

# 颜色定义
COLORS = [0x0000, 0xF800, 0x07E0, 0x001F, 0xFFE0, 0xF81F, 0x07FF, 0xFFFF]

# 方块形状
SHAPES = [
    [[1, 1, 1, 1]],  # I
    [[1, 1, 1], [0, 1, 0]],  # T
    [[1, 1, 1], [1, 0, 0]],  # L
    [[1, 1, 1], [0, 0, 1]],  # J
    [[1, 1], [1, 1]],  # O
    [[0, 1, 1], [1, 1, 0]],  # S
    [[1, 1, 0], [0, 1, 1]],  # Z
]

# 绘制方块
def draw_block(x, y, color):
    display.fill_rect(x * BLOCK_SIZE, y * BLOCK_SIZE, BLOCK_SIZE, BLOCK_SIZE, color)

# 绘制游戏区域
def draw_grid(grid):
    for y in range(GRID_HEIGHT):
        for x in range(GRID_WIDTH):
            if grid[y][x]:
                draw_block(x, y, COLORS[grid[y][x]])

# 检测碰撞
def check_collision(grid, shape, x, y):
    for sy, row in enumerate(shape):
        for sx, block in enumerate(row):
            if block and (y + sy >= GRID_HEIGHT or x + sx < 0 or x + sx >= GRID_WIDTH or grid[y + sy][x + sx]):
                return True
    return False

# 合并方块到游戏区域
def merge_shape(grid, shape, x, y, color):
    for sy, row in enumerate(shape):
        for sx, block in enumerate(row):
            if block:
                grid[y + sy][x + sx] = color

# 消除完整行
def clear_lines(grid):
    lines_cleared = 0
    for y in range(GRID_HEIGHT):
        if all(grid[y]):
            del grid[y]
            grid.insert(0, [0] * GRID_WIDTH)
            lines_cleared += 1
    return lines_cleared

# 摇杆归一化
def normalize_joystick(x, y):
    # 将摇杆值映射到 [-1, 1] 范围
    x_norm = (x - 2048) / 2048
    y_norm = (y - 2048) / 2048
    return x_norm, y_norm

# 俄罗斯方块游戏类
class TetrisGame:
    def __init__(self):
        self.grid = [[0] * GRID_WIDTH for _ in range(GRID_HEIGHT)]
        self.current_shape = random.choice(SHAPES)
        self.current_x = GRID_WIDTH // 2 - len(self.current_shape[0]) // 2
        self.current_y = 0
        self.current_color = random.randint(1, len(COLORS) - 1)
        self.score = 0
        self.last_joystick_state = None  # 记录上一次摇杆状态
        self.last_move_time = time.ticks_ms()  # 记录上一次移动时间
        self.move_interval = 100  # 移动间隔（单位：毫秒）

    def draw(self):
        display.clear()  # 清屏
        draw_grid(self.grid)  # 绘制网格
        for y, row in enumerate(self.current_shape):
            for x, block in enumerate(row):
                if block:
                    draw_block(self.current_x + x, self.current_y + y, COLORS[self.current_color])
        display.show()  # 刷新显示

    def update(self):
        if not check_collision(self.grid, self.current_shape, self.current_x, self.current_y + 1):
            self.current_y += 1
            return True
        else:
            merge_shape(self.grid, self.current_shape, self.current_x, self.current_y, self.current_color)
            self.score += clear_lines(self.grid)
            self.current_shape = random.choice(SHAPES)
            self.current_x = GRID_WIDTH // 2 - len(self.current_shape[0]) // 2
            self.current_y = 0
            self.current_color = random.randint(1, len(COLORS) - 1)
            return not check_collision(self.grid, self.current_shape, self.current_x, self.current_y)

    # 根据摇杆输入移动方块
    def handle_joystick(self, x_norm, y_norm):
        # 检测摇杆状态变化
        current_joystick_state = (x_norm, y_norm)
        if current_joystick_state != self.last_joystick_state:
            self.last_joystick_state = current_joystick_state

            # X 轴控制左右移动
            if x_norm < -0.5:  # 左移
                self.move_left()
            elif x_norm > 0.5:  # 右移
                self.move_right()
            # Y 轴控制下移和旋转
            if y_norm < -0.5:  # 快速下移
                self.move_down_fast()
            elif y_norm > 0.5:  # 旋转
                self.rotate()

    # 左移
    def move_left(self):
        if time.ticks_diff(time.ticks_ms(), self.last_move_time) > self.move_interval:
            if not check_collision(self.grid, self.current_shape, self.current_x - 1, self.current_y):
                self.current_x -= 1
            self.last_move_time = time.ticks_ms()

    # 右移
    def move_right(self):
        if time.ticks_diff(time.ticks_ms(), self.last_move_time) > self.move_interval:
            if not check_collision(self.grid, self.current_shape, self.current_x + 1, self.current_y):
                self.current_x += 1
            self.last_move_time = time.ticks_ms()

    # 快速下移
    def move_down_fast(self):
        if time.ticks_diff(time.ticks_ms(), self.last_move_time) > 200:  # 下移速度稍慢
            if not check_collision(self.grid, self.current_shape, self.current_x, self.current_y + 1):
                self.current_y += 1
            self.last_move_time = time.ticks_ms()

    # 旋转方块
    def rotate(self):
        rotated_shape = list(zip(*self.current_shape[::-1]))  # 将方块矩阵顺时针旋转 90 度
        rotated_shape = [list(row) for row in rotated_shape]  # 将元组转换为列表
        if not check_collision(self.grid, rotated_shape, self.current_x, self.current_y):
            self.current_shape = rotated_shape

# 主程序
def main():
    game = TetrisGame()
    while True:
        x, y = JOYSTICK_X.read(), JOYSTICK_Y.read()  # 读取摇杆输入
        x_norm, y_norm = normalize_joystick(x, y)  # 归一化处理
        game.handle_joystick(x_norm, y_norm)  # 处理摇杆输入

        if not game.update():
            break
        game.draw()
        time.sleep(0.05)  # 整体游戏速度控制

    # 游戏结束显示
    font.text(display, "游戏结束", 64, 100, color=0xF800, bg_color=0x0000, show=True, clear=True)
    font.text(display, f"得分: {game.score}", 64, 120, color=0xF800, bg_color=0x0000, show=True, clear=True)

# 启动游戏
main()
