require("dotenv").config();
const express = require("express");
const mongoose = require("mongoose");
const cors = require("cors");

const app = express();

app.use(cors());
app.use(express.json());
const MONGO_URI = process.env.MONGO_URI;

mongoose
  .connect(MONGO_URI)
  .then(() => console.log("Connected to MongoDB"))
  .catch((err) => console.error("MongoDB Error:", err));

function handleMongoError(err, res) {
  if (err.code === 11000 && err.keyPattern?.email) {
    return res.status(400).json({
      error: "Email đã bị trùng",
      field: "email",
    });
  }

  return res.status(400).json({
    error: err.message,
  });
}

const UserSchema = new mongoose.Schema(
  {
    name: {
      type: String,
      required: [true, "Tên không được để trống"],
      minlength: [2, "Tên phải có ít nhất 2 ký tự"],
      trim: true,
    },
    age: {
      type: Number,
      required: [true, "Tuổi không được để trống"],
      min: [0, "Tuổi phải >= 0"],
      validate: {
        validator: Number.isInteger,
        message: "Tuổi phải là số nguyên",
      },
    },
    email: {
      type: String,
      required: [true, "Email không được để trống"],
      match: [/^\S+@\S+\.\S+$/, "Email không hợp lệ"],
      unique: true,
      trim: true,
      lowercase: true,
    },
    address: {
      type: String,
      trim: true,
      default: "",
    },
  },
  { timestamps: true }
);

const User = mongoose.model("User", UserSchema);

function normalizeUserInput(body) {
  const data = {};

  if (body.name !== undefined) data.name = String(body.name).trim();
  if (body.email !== undefined) data.email = String(body.email).trim().toLowerCase();
  if (body.address !== undefined) data.address = String(body.address).trim();

  if (body.age !== undefined) {
    const ageNumber = Number(body.age);
    data.age = ageNumber;
  }

  return data;
}

app.get("/api/users", async (req, res) => {
  try {
    let page = parseInt(req.query.page) || 1;
    let limit = parseInt(req.query.limit) || 5;
    const search = (req.query.search || "").trim();

    if (page < 1) page = 1;
    if (![3, 5, 10].includes(limit)) limit = 5;
    // tim kiem bang regex
    const filter = search
      ? {
          $or: [
            { name: { $regex: search, $options: "i" } },
            { email: { $regex: search, $options: "i" } },
            { address: { $regex: search, $options: "i" } },
          ],
        }
      : {};

    const skip = (page - 1) * limit;

    const [users, total] = await Promise.all([
      User.find(filter).sort({ createdAt: -1 }).skip(skip).limit(limit),
      User.countDocuments(filter),
    ]);

    const totalPages = Math.ceil(total / limit);

    res.json({
      page,
      limit,
      total,
      totalPages,
      data: users,
    });
  } catch (err) {
    res.status(500).json({ error: err.message });
  }
});

app.post("/api/users", async (req, res) => {
  try {
    const data = normalizeUserInput(req.body);
    const newUser = await User.create(data);

    res.status(201).json({
      message: "Tạo người dùng thành công",
      data: newUser,
    });
  } catch (err) {
    handleMongoError(err, res);
  }
});

app.put("/api/users/:id", async (req, res) => {
  try {
    const { id } = req.params;

    if (!mongoose.Types.ObjectId.isValid(id)) {
      return res.status(400).json({ error: "ID không hợp lệ" });
    }

    const data = normalizeUserInput(req.body);

    const updatedUser = await User.findByIdAndUpdate(id, data, {
      new: true,
      runValidators: true,
    });

    if (!updatedUser) {
      return res.status(404).json({ error: "Không tìm thấy người dùng" });
    }

    res.json({
      message: "Cập nhật người dùng thành công",
      data: updatedUser,
    });
  } catch (err) {
    handleMongoError(err, res);
  }
});

app.delete("/api/users/:id", async (req, res) => {
  try {
    const { id } = req.params;

    if (!mongoose.Types.ObjectId.isValid(id)) {
      return res.status(400).json({ error: "ID không hợp lệ" });
    }

    const deletedUser = await User.findByIdAndDelete(id);

    if (!deletedUser) {
      return res.status(404).json({ error: "Không tìm thấy người dùng" });
    }

    res.json({ message: "Xóa người dùng thành công" });
  } catch (err) {
    res.status(400).json({ error: err.message });
  }
});

app.listen(3001, () => {
  console.log("Server running on http://localhost:3001");
});