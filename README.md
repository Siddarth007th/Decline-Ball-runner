import pygame
import sys
import random
import math
import os

class Obstacle:
    def __init__(self, x, y):
        self.x = x
        self.y = y
        self.shape_type = random.choice(['square', 'rectangle', 'triangle'])

        if self.shape_type == 'square':
            size = 30
            self.rect = pygame.Rect(x, y - size, size, size)
        elif self.shape_type == 'rectangle':
            width, height = 45, 25
            self.rect = pygame.Rect(x, y - height, width, height)
        elif self.shape_type == 'triangle':
            size = 35
            self.rect = pygame.Rect(x, y - size, size, size)

    def draw(self, surface, color):
        if self.shape_type in ['square', 'rectangle']:
            pygame.draw.rect(surface, color, self.rect)
        elif self.shape_type == 'triangle':
            p1 = self.rect.bottomleft
            p2 = self.rect.bottomright
            p3 = (self.rect.centerx, self.rect.top)
            pygame.draw.polygon(surface, color, [p1, p2, p3])

    def update_pos(self, scroll_speed, ground_y_func):
        self.x -= scroll_speed
        self.rect.x = int(self.x)
        self.rect.bottom = int(ground_y_func(self.x))


class Game:
    def __init__(self):
        pygame.init()
        self.width, self.height = 800, 600
        self.screen = pygame.display.set_mode((self.width, self.height))
        pygame.display.set_caption("Decline Runner")
        self.clock = pygame.time.Clock()

        self.white = (255, 255, 255)
        self.blue = (0, 100, 255)
        self.red = (255, 50, 50)
        self.black = (0, 0, 0)

        self.font = pygame.font.SysFont(None, 48)
        self.small_font = pygame.font.SysFont(None, 36)

        self.slope_angle = math.radians(25)  # Steeper slope
        self.ground_y_start = 100
        self.base_scroll_speed = 3
        self.max_scroll_speed = 8

        self.player_radius = 20
        self.gravity = 0.5
        self.jump_strength = -12

        self.obstacle_gap = 350
        self.high_score = self.load_high_score()

        self.game_state = 'menu'
        self.reset_game()

    def get_ground_y(self, x_pos):
        return self.ground_y_start + x_pos * math.tan(self.slope_angle)

    def load_high_score(self):
        if os.path.exists("highscore.txt"):
            with open("highscore.txt", "r") as f:
                try:
                    return int(f.read())
                except (ValueError, IOError):
                    return 0
        return 0

    def save_high_score(self):
        try:
            with open("highscore.txt", "w") as f:
                f.write(str(self.high_score))
        except IOError:
            print("Error: Could not save high score.")

    def reset_game(self):
        self.player_x = self.width // 3  # More central position
        self.player_y = self.get_ground_y(self.player_x)
        self.player_vel_y = 0
        self.on_ground = True
        self.score = 0
        self.scroll_speed = self.base_scroll_speed
        self.obstacles = []

        last_x = self.width
        for _ in range(5):
            spawn_x = last_x + self.obstacle_gap + random.randint(50, 400)
            spawn_y = self.get_ground_y(spawn_x)
            self.obstacles.append(Obstacle(spawn_x, spawn_y))
            last_x = spawn_x

    def run(self):
        while True:
            self.handle_events()
            self.update()
            self.draw()
            self.clock.tick(60)

    def handle_events(self):
        for event in pygame.event.get():
            if event.type == pygame.QUIT:
                self.save_high_score()
                pygame.quit()
                sys.exit()
            if event.type == pygame.KEYDOWN:
                if self.game_state == 'playing':
                    if event.key == pygame.K_SPACE and self.on_ground:
                        self.player_vel_y = self.jump_strength
                        self.on_ground = False
                elif self.game_state in ('menu', 'game_over'):
                    if event.key == pygame.K_RETURN:
                        self.reset_game()
                        self.game_state = 'playing'

    def update(self):
        if self.game_state != 'playing':
            return

        self.scroll_speed = min(self.max_scroll_speed, self.base_scroll_speed + self.score // 5)

        self.player_vel_y += self.gravity
        self.player_y += self.player_vel_y

        ground_y = self.get_ground_y(self.player_x)

        if self.player_y >= ground_y - self.player_radius:
            self.player_y = ground_y - self.player_radius
            self.player_vel_y = 0
            self.on_ground = True

        for obstacle in self.obstacles:
            obstacle.update_pos(self.scroll_speed, self.get_ground_y)

        if self.obstacles and self.obstacles[0].rect.right < 0:
            self.obstacles.pop(0)
            last_x = self.obstacles[-1].x
            spawn_x = last_x + self.obstacle_gap + random.randint(50, 400)
            spawn_y = self.get_ground_y(spawn_x)
            self.obstacles.append(Obstacle(spawn_x, spawn_y))
            self.score += 1

        player_rect = pygame.Rect(self.player_x - self.player_radius, self.player_y - self.player_radius,
                                  self.player_radius * 2, self.player_radius * 2)
        for obstacle in self.obstacles:
            if player_rect.colliderect(obstacle.rect):
                self.game_state = 'game_over'
                if self.score > self.high_score:
                    self.high_score = self.score

    def draw_text(self, text, font, x, y, color, center=False):
        label = font.render(text, True, color)
        rect = label.get_rect()
        if center:
            rect.center = (x, y)
        else:
            rect.topleft = (x, y)
        self.screen.blit(label, rect)

    def draw(self):
        self.screen.fill(self.white)

        start_pos = (0, self.get_ground_y(0))
        end_pos = (self.width, self.get_ground_y(self.width))
        pygame.draw.line(self.screen, self.black, start_pos, end_pos, 2)

        if self.game_state == 'playing':
            pygame.draw.circle(self.screen, self.blue, (int(self.player_x), int(self.player_y)), self.player_radius)
            for obstacle in self.obstacles:
                obstacle.draw(self.screen, self.red)

            self.draw_text(f"Score: {self.score}", self.small_font, 10, 10, self.black)
            self.draw_text(f"High Score: {self.high_score}", self.small_font, 10, 40, self.black)

        if self.game_state == 'menu':
            self.draw_text("Decline Runner", self.font, self.width / 2, self.height / 4, self.black, center=True)
            self.draw_text("Press ENTER to Start", self.small_font, self.width / 2, self.height / 2, self.black, center=True)
            self.draw_text(f"High Score: {self.high_score}", self.small_font, self.width / 2, self.height / 2 + 40, self.black, center=True)

        elif self.game_state == 'game_over':
            self.draw_text("Game Over", self.font, self.width / 2, self.height / 4, self.black, center=True)
            self.draw_text(f"Your Score: {self.score}", self.small_font, self.width / 2, self.height / 2 - 40, self.black, center=True)
            self.draw_text(f"High Score: {self.high_score}", self.small_font, self.width / 2, self.height / 2, self.black, center=True)
            self.draw_text("Press ENTER to Restart", self.small_font, self.width / 2, self.height / 2 + 60, self.black, center=True)

        pygame.display.flip()


if __name__ == "__main__":
    game = Game()
    game.run()
