import json, os

SAVE_FILE = "savegame.json"

def save_game(self):
    data = {
        "bus_pos": [self.bus_np.getX(), self.bus_np.getY(), self.bus_np.getZ()],
        "bus_h": self.bus_np.getH(),
        "passengers": self.passengers
    }
    with open(SAVE_FILE, "w") as f:
        json.dump(data, f)
    print("✅ Game Saved")

def load_game(self):
    if not os.path.exists(SAVE_FILE):
        print("❌ No save file")
        return
    with open(SAVE_FILE) as f:
        data = json.load(f)

    self.bus_np.setPos(*data["bus_pos"])
    self.bus_np.setH(data["bus_h"])
    self.passengers = data["passengers"]
    print("✅ Game Loaded")
