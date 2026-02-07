from direct.showbase.ShowBase import ShowBase
from panda3d.core import *
from panda3d.bullet import *
from direct.task import Task
from direct.gui.OnscreenText import OnscreenText
import random, json, os

SAVE_FILE = "savegame.json"

class BusGame(ShowBase):
    def __init__(self):
        super().__init__()
        self.disableMouse()

        # ---------- PHYSICS ----------
        self.world = BulletWorld()
        self.world.setGravity(Vec3(0, 0, -9.8))

        # ---------- LIGHT ----------
        sun = DirectionalLight("sun")
        sun.setDirection(Vec3(-1, -1, -2))
        self.render.setLight(self.render.attachNewNode(sun))

        # ---------- CITY ----------
        self.create_city()

        # ---------- BUS ----------
        self.bus = self.loader.loadModel("models/box")
        self.bus.setScale(1.5, 4, 1.5)
        self.bus.setColor(1, 0, 0, 1)

        shape = BulletBoxShape(Vec3(1.5, 4, 1.5))
        body = BulletRigidBodyNode("Bus")
        body.setMass(900)
        body.addShape(shape)

        self.bus_np = self.render.attachNewNode(body)
        self.bus_np.setPos(0, 0, 2)
        self.bus.reparentTo(self.bus_np)
        self.world.attachRigidBody(body)

        # ---------- ROUTES & STOPS ----------
        self.stops = [
            Vec3(10, 30, 1),
            Vec3(-20, 80, 1),
            Vec3(40, 120, 1)
        ]
        self.current_stop = 0
        self.passengers = 0

        self.stop_marker = self.loader.loadModel("models/box")
        self.stop_marker.setScale(1, 1, 2)
        self.stop_marker.setColor(1, 1, 0, 1)
        self.stop_marker.reparentTo(self.render)
        self.update_stop_marker()

        # ---------- TRAFFIC ----------
        self.cars = []
        for i in range(5):
            car = self.loader.loadModel("models/box")
            car.setScale(1, 2, 1)
            car.setColor(0, 0, 1, 1)
            car.setPos(random.randint(-30, 30), random.randint(-50, 150), 1)
            car.reparentTo(self.render)
            self.cars.append(car)

        # ---------- TRAFFIC LIGHT ----------
        self.light_timer = 0
        self.green_light = True

        # ---------- CONTROLS ----------
        self.keys = {"w":0,"s":0,"a":0,"d":0}
        for k in self.keys:
            self.accept(k, self.set_key, [k, 1])
            self.accept(k+"-up", self.set_key, [k, 0])

        self.accept("f5", self.save_game)
        self.accept("f9", self.load_game)

        # ---------- UI ----------
        self.speed_text = OnscreenText(pos=(-1.3, 0.9), scale=0.05)
        self.info_text = OnscreenText(pos=(-1.3, 0.85), scale=0.05)

        # ---------- SOUND ----------
        self.engine = self.loader.loadSfx("models/audio/sfx/GUI_rollover.wav")
        self.engine.setLoop(True)

        self.taskMgr.add(self.update, "update")

    # ---------------- CITY ----------------
    def create_city(self):
        for x in range(-3, 4):
            road = CardMaker("road")
            road.setFrame(-5, 5, -200, 200)
            r = self.render.attachNewNode(road.generate())
            r.setP(-90)
            r.setX(x * 15)
            r.setColor(0.2, 0.2, 0.2, 1)

    # ---------------- INPUT ----------------
    def set_key(self, key, val):
        self.keys[key] = val

    # ---------------- SAVE / LOAD ----------------
    def save_game(self):
        data = {
            "bus_pos": list(self.bus_np.getPos()),
            "passengers": self.passengers,
            "stop": self.current_stop
        }
        with open(SAVE_FILE, "w") as f:
            json.dump(data, f)
        print("Game Saved")

    def load_game(self):
        if not os.path.exists(SAVE_FILE):
            return
        with open(SAVE_FILE) as f:
            data = json.load(f)
        self.bus_np.setPos(Vec3(*data["bus_pos"]))
        self.passengers = data["passengers"]
        self.current_stop = data["stop"]
        self.update_stop_marker()
        print("Game Loaded")

    # ---------------- STOPS ----------------
    def update_stop_marker(self):
        self.stop_marker.setPos(self.stops[self.current_stop])

    # ---------------- UPDATE ----------------
    def update(self, task):
        dt = globalClock.getDt()
        self.world.doPhysics(dt)

        force = Vec3(0,0,0)

        if self.keys["w"]:
            force.y = 3000
            if self.engine.status() != self.engine.PLAYING:
                self.engine.play()
        if self.keys["s"]:
            force.y = -2000
        if self.keys["a"]:
            self.bus_np.setH(self.bus_np.getH() + 70 * dt)
        if self.keys["d"]:
            self.bus_np.setH(self.bus_np.getH() - 70 * dt)

        self.bus_np.applyCentralForce(self.bus_np.getQuat().xform(force))

        # ---------- CAMERA ----------
        cam_pos = self.bus_np.getPos() + Vec3(0, -35, 14)
        self.camera.setPos(cam_pos)
        self.camera.lookAt(self.bus_np)

        # ---------- TRAFFIC AI ----------
        self.light_timer += dt
        if self.light_timer > 5:
            self.green_light = not self.green_light
            self.light_timer = 0

        for car in self.cars:
            if self.green_light:
                car.setY(car, 12 * dt)
            if car.getY() > 200:
                car.setY(-200)

        # ---------- PASSENGERS ----------
        if (self.bus_np.getPos() - self.stops[self.current_stop]).length() < 5:
            self.passengers += random.randint(1, 4)
            self.current_stop = (self.current_stop + 1) % len(self.stops)
            self.update_stop_marker()

        # ---------- UI ----------
        speed = int(self.bus_np.node().getLinearVelocity().length())
        self.speed_text.setText(f"Speed: {speed} km/h")
        self.info_text.setText(
            f"Passengers: {self.passengers} | Stop: {self.current_stop+1}"
        )

        return Task.cont


game = BusGame()
game.run()
