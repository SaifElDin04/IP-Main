# Community Module Documentation

This document provides a comprehensive overview of the Community module in the Reddit Clone application. It covers the database schema, backend architecture, frontend architecture, administrative logic, and file summaries.

---

## Table of Contents

1. [Overview](#overview)
2. [Database Schema](#database-schema)
3. [Backend Architecture](#backend-architecture)
   - [Routes](#routes)
   - [Controllers](#controllers)
   - [Services](#services)
4. [Frontend Architecture](#frontend-architecture)
   - [API Services](#api-services)
   - [Pages](#pages)
   - [Components](#components)
5. [Administrative Logic](#administrative-logic)
   - [Kick Member](#kick-member)
   - [Promote to Admin](#promote-to-admin)
   - [Permissions Matrix](#permissions-matrix)
6. [File Summary](#file-summary)

---

## Overview

The Community module enables users to create and manage communities (subreddits). Each community has a creator, members, and optionally admins. Communities serve as containers for posts and facilitate user engagement around specific topics.

**Key Features:**
- Create communities with custom names and descriptions
- Join/leave communities
- Upload avatar and banner images
- Member management (kick, promote)
- Search functionality for communities

---

## Database Schema

### `backend/models/Community.js`

The Community model defines the structure for community documents in MongoDB.

```javascript
const mongoose = require("mongoose");

const communitySchema = new mongoose.Schema(
  {
    name: {
      type: String,
      required: [true, "Community name is required"],
      unique: true,
      trim: true,
      minlength: 3,
      maxlength: 21,
      match: [/^[A-Za-z0-9_]+$/, "Community name can only contain letters, numbers, and underscores"]
    },
    description: {
      type: String,
      maxlength: 500,
      default: ""
    },
    creator: {
      type: mongoose.Schema.Types.ObjectId,
      ref: "User",
      required: true
    },
    members: [
      {
        type: mongoose.Schema.Types.ObjectId,
        ref: "User"
      }
    ],
    admins: [
      {
        type: mongoose.Schema.Types.ObjectId,
        ref: "User"
      }
    ],
    avatar: {
      type: String,
      default: ""
    },
    banner: {
      type: String,
      default: ""
    }
  },
  {
    timestamps: true
  }
);

module.exports = mongoose.model("Community", communitySchema);
```

**Schema Fields:**

| Field | Type | Description | Constraints |
|-------|------|-------------|-------------|
| `name` | String | Community name (displayed as r/name) | Required, unique, 3-21 chars, alphanumeric + underscore only |
| `description` | String | Community description | Max 500 chars |
| `creator` | ObjectId | User who created the community | Required, references User model |
| `members` | Array[ObjectId] | List of community members | References User model |
| `admins` | Array[ObjectId] | List of admin users | References User model |
| `avatar` | String | URL to community avatar image | Default empty string |
| `banner` | String | URL to community banner image | Default empty string |

**Important Behaviors:**
- `creator` is automatically added to `members` when creating a community
- `creator` is NOT automatically added to `admins`
- Only the `creator` can promote members to admin status
- Both `creator` and `admins` can manage the community profile

---

## Backend Architecture

### Routes

### `backend/routes/communities.js`

Defines all API endpoints for community operations.

```javascript
const express = require("express");
const router = express.Router();

const {
    createCommunity,
    getAllCommunities,
    getCommunityById,
    joinCommunity,
    leaveCommunity,
    searchCommunities,
    uploadCommunityAvatar,
    uploadCommunityBanner,
    kickMember,
    promoteToAdmin
} = require("../controllers/communityController");
const { protect } = require("../middleware/auth");
const upload = require("../middleware/upload");

// Public routes
router.get("/search", searchCommunities);
router.get("/", getAllCommunities);
router.get("/:id", getCommunityById);

// Protected routes (require authentication)
router.post("/", protect, createCommunity);
router.post("/:id/join", protect, joinCommunity);
router.post("/:id/leave", protect, leaveCommunity);
router.post("/:id/avatar", protect, upload.single("image"), uploadCommunityAvatar);
router.post("/:id/banner", protect, upload.single("image"), uploadCommunityBanner);
router.post("/:id/kick/:userId", protect, kickMember);
router.post("/:id/promote/:userId", protect, promoteToAdmin);

module.exports = router;
```

**API Endpoints:**

| Method | Endpoint | Description | Auth Required |
|--------|----------|-------------|---------------|
| GET | `/api/communities` | Get all communities | No |
| GET | `/api/communities/:id` | Get community by ID | No |
| GET | `/api/communities/search?q=query` | Search communities | No |
| POST | `/api/communities` | Create a new community | Yes |
| POST | `/api/communities/:id/join` | Join a community | Yes |
| POST | `/api/communities/:id/leave` | Leave a community | Yes |
| POST | `/api/communities/:id/avatar` | Upload community avatar | Yes |
| POST | `/api/communities/:id/banner` | Upload community banner | Yes |
| POST | `/api/communities/:id/kick/:userId` | Kick a member | Yes |
| POST | `/api/communities/:id/promote/:userId` | Promote member to admin | Yes |

---

### Controllers

### `backend/controllers/communityController.js`

Controllers handle HTTP requests/responses, validation, and call service methods.

```javascript
const communityService = require('../services/communityService');
const { uploadBufferToCloudinary } = require("../utils/cloudinaryUpload");

const createCommunity = async (req, res) => {
    try {
        const { name, description } = req.body;
        const creatorId = req.user._id;
        const community = await communityService.createCommunity({ name, description, creatorId });
        return res.status(201).json({
            success: true,
            message: "Community created successfully",
            data: community,
        });
    } catch (error) {
        return res.status(400).json({
            success: false,
            message: error.message,
        });
    }
};

const getAllCommunities = async (req, res) => {
    try {
        const communities = await communityService.getAllCommunities();
        return res.status(200).json({
            success: true,
            data: communities,
        });
    } catch (error) {
        return res.status(400).json({
            success: false,
            message: error.message,
        });
    }
};

const getCommunityById = async (req, res) => {
    try {
        const communityId = req.params.id;
        const community = await communityService.getCommunityById(communityId);
        return res.status(200).json({
            success: true,
            data: community,
        });
    } catch (error) {
        return res.status(400).json({
            success: false,
            message: error.message,
        });
    }
};

const joinCommunity = async (req, res) => {
    try {
        const communityId = req.params.id;
        const userId = req.user._id;
        const community = await communityService.joinCommunity(communityId, userId);
        return res.status(200).json({
            success: true,
            message: "Joined community successfully",
            data: community,
        });
    } catch (error) {
        return res.status(400).json({
            success: false,
            message: error.message,
        });
    }
};

const leaveCommunity = async (req, res) => {
    try {
        const communityId = req.params.id;
        const userId = req.user._id;
        const community = await communityService.leaveCommunity(communityId, userId);
        return res.status(200).json({
            success: true,
            message: "Left community successfully",
            data: community,
        });
    } catch (error) {
        return res.status(400).json({
            success: false,
            message: error.message,
        });
    }
};

const searchCommunities = async (req, res) => {
    try {
        const query = req.query.q;
        const communities = await communityService.searchCommunities(query);
        
        return res.status(200).json({
            success: true,
            data: { communities },
        });
    } catch (error) {
        return res.status(500).json({
            success: false,
            message: "Search failed",
        });
    }
};

const uploadCommunityAvatar = async (req, res) => {
    try {
        if (!req.file) {
            return res.status(400).json({
                success: false,
                message: "Image file is required"
            });
        }

        const communityId = req.params.id;
        const result = await uploadBufferToCloudinary(
            req.file.buffer,
            "reddit-clone/community-avatars",
            `community_avatar_${communityId}`
        );

        const community = await communityService.updateCommunityProfile(communityId, req.user._id, {
            avatar: result.secure_url
        });

        return res.status(200).json({
            success: true,
            message: "Community avatar uploaded successfully",
            data: community
        });
    } catch (error) {
        return res.status(error.statusCode || 400).json({
            success: false,
            message: error.message || "Community avatar upload failed"
        });
    }
};

const uploadCommunityBanner = async (req, res) => {
    try {
        if (!req.file) {
            return res.status(400).json({
                success: false,
                message: "Image file is required"
            });
        }

        const communityId = req.params.id;
        const result = await uploadBufferToCloudinary(
            req.file.buffer,
            "reddit-clone/community-banners",
            `community_banner_${communityId}`
        );

        const community = await communityService.updateCommunityProfile(communityId, req.user._id, {
            banner: result.secure_url
        });

        return res.status(200).json({
            success: true,
            message: "Community banner uploaded successfully",
            data: community
        });
    } catch (error) {
        return res.status(error.statusCode || 400).json({
            success: false,
            message: error.message || "Community banner upload failed"
        });
    }
};

const kickMember = async (req, res) => {
    try {
        const communityId = req.params.id;
        const targetUserId = req.params.userId;
        const adminId = req.user._id;
        const community = await communityService.kickMember(communityId, adminId, targetUserId);
        return res.status(200).json({
            success: true,
            message: "Member kicked successfully",
            data: community,
        });
    } catch (error) {
        return res.status(error.statusCode || 400).json({
            success: false,
            message: error.message,
        });
    }
};

const promoteToAdmin = async (req, res) => {
    try {
        const communityId = req.params.id;
        const targetUserId = req.params.userId;
        const ownerId = req.user._id;
        const community = await communityService.promoteToAdmin(communityId, ownerId, targetUserId);
        return res.status(200).json({
            success: true,
            message: "Member promoted to admin successfully",
            data: community,
        });
    } catch (error) {
        return res.status(error.statusCode || 400).json({
            success: false,
            message: error.message,
        });
    }
};

module.exports = {
    createCommunity,
    getAllCommunities,
    getCommunityById,
    joinCommunity,
    leaveCommunity,
    searchCommunities,
    uploadCommunityAvatar,
    uploadCommunityBanner,
    kickMember,
    promoteToAdmin
};
```

---

### Services

### `backend/services/communityService.js`

Services contain the core business logic and interact directly with the database.

```javascript
const Community = require('../models/Community');

const createError = (message, statusCode) => {
    const error = new Error(message);
    error.statusCode = statusCode;
    return error;
};

const createCommunity = async ({ name, description, creatorId }) => {
    const existingCommunity = await Community.findOne({ name });
    if (existingCommunity) {
        throw createError("Community with this name already exists", 400);
    }

    const community = await Community.create({
        name,
        description,
        creator: creatorId,
        members: [creatorId],  // Creator is automatically added as a member
        admins: []              // Creator is NOT automatically an admin
    });
    return community;
};

const getAllCommunities = async () => {
    return await Community.find().populate("creator", "username");
};

const getCommunityById = async (communityId) => {
    const community = await Community.findById(communityId)
        .populate("creator", "username")
        .populate("members", "username avatar")
        .populate("admins", "username avatar");
    if (!community) {
        throw createError("Community not found", 404);
    }
    return community;
};

const joinCommunity = async (communityId, userId) => {
    const community = await Community.findById(communityId);
    if (!community) {
        throw createError("Community not found", 404);
    }

    if (community.members.includes(userId)) {
        return community;  // Already a member
    }

    community.members.push(userId);
    await community.save();
    return community;
};

const leaveCommunity = async (communityId, userId) => {
    const community = await Community.findById(communityId);
    if (!community) {
        throw createError("Community not found", 404);
    }

    community.members = community.members.filter(id => id.toString() !== userId.toString());
    community.admins = community.admins.filter(id => id.toString() !== userId.toString());
    await community.save();
    return community;
};

const searchCommunities = async (query) => {
    if (!query) return [];
    const communities = await Community.find({
        name: { $regex: query, $options: 'i' }
    })
    .select('_id name avatar members')
    .limit(5);

    return communities;
};

const updateCommunityProfile = async (communityId, userId, updateData) => {
    const community = await Community.findById(communityId);
    if (!community) {
        throw createError("Community not found", 404);
    }

    const isAdmin = community.admins.some(adminId => adminId.toString() === userId.toString());
    const isCreator = community.creator.toString() === userId.toString();

    if (!isCreator && !isAdmin) {
        throw createError("Only the creator or admins can update the community profile", 403);
    }

    Object.assign(community, updateData);
    await community.save();
    return community;
};

const kickMember = async (communityId, adminId, targetUserId) => {
    const community = await Community.findById(communityId);
    if (!community) {
        throw createError("Community not found", 404);
    }

    const isAdmin = community.admins.some(id => id.toString() === adminId.toString());
    const isCreator = community.creator.toString() === adminId.toString();

    if (!isCreator && !isAdmin) {
        throw createError("Only the creator or admins can kick members", 403);
    }

    if (targetUserId.toString() === community.creator.toString()) {
        throw createError("Cannot kick the community creator", 400);
    }

    const targetIsAdmin = community.admins.some(id => id.toString() === targetUserId.toString());
    if (targetIsAdmin && !isCreator) {
        throw createError("Only the creator can kick other admins", 403);
    }

    community.members = community.members.filter(id => id.toString() !== targetUserId.toString());
    community.admins = community.admins.filter(id => id.toString() !== targetUserId.toString());
    await community.save();
    return community;
};

const promoteToAdmin = async (communityId, ownerId, targetUserId) => {
    const community = await Community.findById(communityId);
    if (!community) {
        throw createError("Community not found", 404);
    }

    if (community.creator.toString() !== ownerId.toString()) {
        throw createError("Only the creator can promote members to admin", 403);
    }

    if (!community.members.some(id => id.toString() === targetUserId.toString())) {
        throw createError("User must be a member of the community", 400);
    }

    if (community.admins.some(id => id.toString() === targetUserId.toString())) {
        return community; // Already an admin
    }

    community.admins.push(targetUserId);
    await community.save();
    return community;
};

module.exports = {
    createCommunity,
    getAllCommunities,
    getCommunityById,
    joinCommunity,
    leaveCommunity,
    searchCommunities,
    updateCommunityProfile,
    kickMember,
    promoteToAdmin
};
```

---

## Frontend Architecture

### API Services

### `frontend/src/services/communityService.js`

Provides a clean interface for making API calls to the backend.

```javascript
import api from './api';

export const communityService = {
    getAllCommunities: () => api.get('/communities'),
    getCommunityById: (communityId) => api.get(`/communities/${communityId}`),
    createCommunity: (communityData) => api.post('/communities', communityData),
    joinCommunity: (communityId) => api.post(`/communities/${communityId}/join`),
    leaveCommunity: (communityId) => api.post(`/communities/${communityId}/leave`),
    searchCommunities: (query) => api.get(`/communities/search?q=${query}`),
    uploadAvatar: (communityId, formData) => api.post(`/communities/${communityId}/avatar`, formData, {
        headers: { 'Content-Type': 'multipart/form-data' }
    }),
    uploadBanner: (communityId, formData) => api.post(`/communities/${communityId}/banner`, formData, {
        headers: { 'Content-Type': 'multipart/form-data' }
    }),
    kickMember: (communityId, userId) => api.post(`/communities/${communityId}/kick/${userId}`),
    promoteToAdmin: (communityId, userId) => api.post(`/communities/${communityId}/promote/${userId}`),
};
```

The `api` instance (from `api.js`) automatically handles:
- Base URL configuration
- JWT token injection from localStorage
- Response interceptors for auth errors

---

### Pages

#### `frontend/src/pages/CommunitiesPage.js`

Displays a list of all communities and allows users to join/leave them.

```javascript
import React, { useState, useEffect } from "react";
import { Typography, Box } from "@mui/material";
import Navbar from "../components/layout/Navbar";
import LeftSidebar from "../components/layout/LeftSidebar";
import RightSidebar from "../components/layout/RightSidebar";
import { useAuth } from "../context/AuthContext";
import CommunityList from "../components/communities/CommunityList";
import { communityService } from "../services/communityService";

function CommunitiesPage() {
    const [communities, setCommunities] = useState([]);
    const { user } = useAuth();

    useEffect(() => {
        fetchCommunities();
    }, []);

    const fetchCommunities = async () => {
        try {
            const response = await communityService.getAllCommunities();
            setCommunities(response.data.data);
        } catch (error) {
            console.error("Error fetching communities:", error);
        }
    };

    const handleJoin = async (communityId) => {
        try {
            await communityService.joinCommunity(communityId);
            fetchCommunities();
        } catch (error) {
            console.error("Error joining community:", error);
        }
    };

    const handleLeave = async (communityId) => {
        try {
            await communityService.leaveCommunity(communityId);
            fetchCommunities();
        } catch (error) {
            console.error("Error leaving community:", error);
        }
    };

    return (
        <Box sx={{ minHeight: "100vh", backgroundColor: "#111111ff", color: "#d7dadc", display: "flex", flexDirection: "column" }}>
            <Navbar />

            <Box sx={{ display: "flex", flexGrow: 1, pt: "56px" }}>
                <LeftSidebar />

                <Box sx={{ flexGrow: 1, overflowY: "auto", display: "flex", justifyContent: "center" }}>
                    <Box sx={{ width: "100%", maxWidth: "900px", px: 2, py: 4 }}>
                        <Box sx={{ mb: 4, display: 'flex', justifyContent: 'space-between', alignItems: 'center' }}>
                            <Typography variant="h4" sx={{ fontWeight: 'bold' }}>
                                Browse Communities
                            </Typography>
                        </Box>
                        <CommunityList 
                            communities={communities} 
                            onJoin={handleJoin} 
                            onLeave={handleLeave} 
                            currentUserId={user?._id || user?.id}
                        />
                    </Box>
                </Box>

                <RightSidebar />
            </Box>
        </Box>
    );
}

export default CommunitiesPage;
```

#### `frontend/src/pages/CommunityDetailPage.js`

Shows detailed community information including posts, members, and admin controls.

```javascript
import React, { useState, useEffect } from "react";
import { useParams } from "react-router-dom";
import { Box, Tabs, Tab, Typography, List, ListItem, ListItemAvatar, Avatar, ListItemText, IconButton, Tooltip, Paper } from "@mui/material";
import PersonRemoveIcon from '@mui/icons-material/PersonRemove';
import AdminPanelSettingsIcon from '@mui/icons-material/AdminPanelSettings';
import Navbar from "../components/layout/Navbar";
import LeftSidebar from "../components/layout/LeftSidebar";
import RightSidebar from "../components/layout/RightSidebar";
import { useAuth } from "../context/AuthContext";
import PostList from "../components/posts/PostList";
import CommunityHeader from "../components/communities/CommunityHeader";
import { communityService } from "../services/communityService";

function CommunityDetailPage() {
    const { communityId } = useParams();
    const [community, setCommunity] = useState(null);
    const [tabValue, setTabValue] = useState(0);
    const { user } = useAuth();

    useEffect(() => {
        fetchCommunity();
    }, [communityId]);

    const fetchCommunity = async () => {
        try {
            const response = await communityService.getCommunityById(communityId);
            setCommunity(response.data.data);
        } catch (error) {
            console.error("Error fetching community:", error);
        }
    };

    const handleJoin = async () => {
        try {
            await communityService.joinCommunity(communityId);
            fetchCommunity();
        } catch (error) {
            console.error("Error joining community:", error);
        }
    };

    const handleLeave = async () => {
        try {
            await communityService.leaveCommunity(communityId);
            fetchCommunity();
        } catch (error) {
            console.error("Error leaving community:", error);
        }
    };

    const handleKick = async (targetUserId) => {
        if (!window.confirm("Are you sure you want to kick this member?")) return;
        try {
            await communityService.kickMember(communityId, targetUserId);
            fetchCommunity();
        } catch (error) {
            console.error("Error kicking member:", error);
            alert(error.response?.data?.message || "Failed to kick member");
        }
    };

    const handlePromote = async (targetUserId) => {
        if (!window.confirm("Are you sure you want to promote this member to admin?")) return;
        try {
            await communityService.promoteToAdmin(communityId, targetUserId);
            fetchCommunity();
        } catch (error) {
            console.error("Error promoting member:", error);
            alert(error.response?.data?.message || "Failed to promote member");
        }
    };

    const handleTabChange = (event, newValue) => {
        setTabValue(newValue);
    };

    const userId = user?._id || user?.id;
    const isMember = community?.members?.some(member => 
        (typeof member === 'string' ? member : member._id) === userId
    );

    const isCreator = community?.creator?._id === userId || community?.creator === userId;
    const currentUserIsAdmin = community?.admins?.some(admin => (admin._id || admin) === userId);
    const canManage = isCreator || currentUserIsAdmin;

    return (
        <Box sx={{ minHeight: "100vh", backgroundColor: "#111111ff", color: "#d7dadc", display: "flex", flexDirection: "column" }}>
            <Navbar />

            <Box sx={{ display: "flex", flexGrow: 1, pt: "56px" }}>
                <LeftSidebar />

                <Box sx={{ flexGrow: 1, overflowY: "auto" }}>
                    <CommunityHeader 
                        community={community} 
                        onJoin={handleJoin} 
                        onLeave={handleLeave}
                        isMember={isMember}
                        currentUserId={userId}
                        onUpdate={fetchCommunity}
                    />

                    <Box sx={{ borderBottom: 1, borderColor: '#343536', bgcolor: '#1A1A1B' }}>
                        <Tabs 
                            value={tabValue} 
                            onChange={handleTabChange} 
                            textColor="inherit"
                            indicatorColor="primary"
                            sx={{
                                px: { xs: 1, md: 2 },
                                '& .MuiTab-root': {
                                    textTransform: 'none',
                                    fontWeight: 700,
                                    fontSize: '14px',
                                    minWidth: 100
                                }
                            }}
                        >
                            <Tab label="Posts" />
                            <Tab label="Members" />
                        </Tabs>
                    </Box>

                    <Box sx={{ display: "flex", justifyContent: "center", alignItems: "flex-start", gap: 3, py: 4, px: 2 }}>
                        <Box sx={{ width: "100%", maxWidth: "700px" }}>
                            {tabValue === 0 && (
                                <PostList communityId={communityId} />
                            )}
                            
                            {tabValue === 1 && (
                                <Paper sx={{ bgcolor: '#1A1A1B', borderRadius: 1, border: '1px solid #343536', overflow: 'hidden' }}>
                                    <Typography variant="h6" sx={{ p: 2, borderBottom: '1px solid #343536', color: '#D7DADC' }}>
                                        Community Members ({community?.members?.length || 0})
                                    </Typography>
                                    <List>
                                        {community?.members?.map((member) => {
                                            const memberId = member._id || member;
                                            const isMemberCreator = community.creator?._id === memberId || community.creator === memberId;
                                            const isMemberAdmin = community.admins?.some(admin => (admin._id || admin) === memberId);
                                            
                                            return (
                                                <ListItem 
                                                    key={memberId}
                                                    divider
                                                    sx={{ borderColor: '#343536' }}
                                                    secondaryAction={
                                                        <Box>
                                                            {isCreator && !isMemberCreator && !isMemberAdmin && (
                                                                <Tooltip title="Promote to Admin">
                                                                    <IconButton onClick={() => handlePromote(memberId)} sx={{ color: '#818384' }}>
                                                                        <AdminPanelSettingsIcon />
                                                                    </IconButton>
                                                                </Tooltip>
                                                            )}
                                                            {canManage && !isMemberCreator && (userId !== memberId) && (isCreator || !isMemberAdmin) && (
                                                                <Tooltip title="Kick Member">
                                                                    <IconButton onClick={() => handleKick(memberId)} sx={{ color: '#ff585b' }}>
                                                                        <PersonRemoveIcon />
                                                                    </IconButton>
                                                                </Tooltip>
                                                            )}
                                                        </Box>
                                                    }
                                                >
                                                    <ListItemAvatar>
                                                        <Avatar src={member.avatar}>{member.username?.charAt(0)}</Avatar>
                                                    </ListItemAvatar>
                                                    <ListItemText 
                                                        primary={
                                                            <Box sx={{ display: 'flex', alignItems: 'center', gap: 1 }}>
                                                                <Typography sx={{ color: '#D7DADC', fontWeight: 600 }}>
                                                                    u/{member.username || 'unknown'}
                                                                </Typography>
                                                                {isMemberCreator && (
                                                                    <Typography variant="caption" sx={{ bgcolor: '#0079D3', color: 'white', px: 1, borderRadius: 1 }}>
                                                                        Owner
                                                                    </Typography>
                                                                )}
                                                                {isMemberAdmin && !isMemberCreator && (
                                                                    <Typography variant="caption" sx={{ bgcolor: '#46D160', color: 'white', px: 1, borderRadius: 1 }}>
                                                                        Admin
                                                                    </Typography>
                                                                )}
                                                            </Box>
                                                        }
                                                    />
                                                </ListItem>
                                            );
                                        })}
                                    </List>
                                </Paper>
                            )}
                        </Box>
                        <RightSidebar />
                    </Box>
                </Box>
            </Box>
        </Box>
    );
}

export default CommunityDetailPage;
```

---

### Components

#### `frontend/src/components/communities/CommunityHeader.jsx`

Displays the community banner, avatar, name, and join/leave button. Also provides upload controls for admins.

```javascript
import React, { useRef, useState } from 'react';
import { 
    Box, 
    Typography, 
    Button, 
    Avatar, 
    Container, 
    Paper,
    Stack,
    IconButton,
    CircularProgress
} from '@mui/material';
import GroupsIcon from '@mui/icons-material/Groups';
import PhotoCameraIcon from '@mui/icons-material/PhotoCamera';
import EditIcon from '@mui/icons-material/Edit';
import { communityService } from '../../services/communityService';

const CommunityHeader = ({ community, onJoin, onLeave, isMember, currentUserId, onUpdate }) => {
    const [uploading, setUploading] = useState({ avatar: false, banner: false });
    const avatarInputRef = useRef(null);
    const bannerInputRef = useRef(null);

    if (!community) return null;

    const isCreator = community.creator === currentUserId || community.creator?._id === currentUserId;
    const isAdmin = community.admins?.some(admin => {
        const adminId = typeof admin === 'string' ? admin : admin._id;
        return adminId === currentUserId;
    });
    const canEdit = isCreator || isAdmin;

    const handleFileChange = async (event, type) => {
        const file = event.target.files[0];
        if (!file) return;

        const formData = new FormData();
        formData.append('image', file);

        setUploading(prev => ({ ...prev, [type]: true }));
        try {
            if (type === 'avatar') {
                await communityService.uploadAvatar(community._id, formData);
            } else {
                await communityService.uploadBanner(community._id, formData);
            }
            if (onUpdate) onUpdate();
        } catch (error) {
            console.error(`Error uploading ${type}:`, error);
            alert(`Failed to upload ${type}`);
        } finally {
            setUploading(prev => ({ ...prev, [type]: false }));
        }
    };

    return (
        <Paper elevation={0} sx={{ mb: 3, overflow: 'hidden', borderRadius: 0, backgroundColor: '#1A1A1B', borderBottom: '1px solid #343536' }}>
            {/* Banner Section */}
            <Box sx={{ height: 180, bgcolor: '#33a8ff', position: 'relative' }}>
                {community.banner && (
                    <img src={community.banner} alt="banner" style={{ width: '100%', height: '100%', objectFit: 'cover' }} />
                )}
                {canEdit && (
                    <>
                        <input type="file" hidden ref={bannerInputRef} onChange={(e) => handleFileChange(e, 'banner')} accept="image/*" />
                        <IconButton
                            sx={{ position: 'absolute', bottom: 10, right: 10, bgcolor: 'rgba(0,0,0,0.5)', '&:hover': { bgcolor: 'rgba(0,0,0,0.7)' }, color: 'white' }}
                            onClick={() => bannerInputRef.current.click()}
                            disabled={uploading.banner}
                        >
                            {uploading.banner ? <CircularProgress size={24} color="inherit" /> : <PhotoCameraIcon />}
                        </IconButton>
                    </>
                )}
            </Box>

            <Container maxWidth="lg">
                <Box sx={{ display: 'flex', alignItems: 'flex-start', mt: -3, pb: 2, px: { xs: 1, md: 2 } }}>
                    {/* Avatar Section */}
                    <Box sx={{ position: 'relative' }}>
                        <Avatar sx={{ width: 80, height: 80, border: '4px solid #1A1A1B', bgcolor: community.avatar ? 'white' : '#0079D3', boxShadow: '0 2px 4px rgba(0,0,0,0.2)' }}>
                            {community.avatar ? (
                                <img src={community.avatar} alt={community.name} width="100%" height="100%" style={{ objectFit: 'cover' }} />
                            ) : (
                                <GroupsIcon sx={{ fontSize: 45 }} />
                            )}
                        </Avatar>
                        {canEdit && (
                            <>
                                <input type="file" hidden ref={avatarInputRef} onChange={(e) => handleFileChange(e, 'avatar')} accept="image/*" />
                                <IconButton size="small" sx={{ position: 'absolute', bottom: 0, right: 0, bgcolor: '#D7DADC', '&:hover': { bgcolor: '#EBEDEF' }, color: '#1A1A1B', border: '2px solid #1A1A1B' }}
                                    onClick={() => avatarInputRef.current.click()} disabled={uploading.avatar}>
                                    {uploading.avatar ? <CircularProgress size={16} color="inherit" /> : <EditIcon sx={{ fontSize: 16 }} />}
                                </IconButton>
                            </>
                        )}
                    </Box>

                    {/* Info Section */}
                    <Box sx={{ ml: 2, mt: 4, flexGrow: 1 }}>
                        <Stack direction="row" justifyContent="space-between" alignItems="center" spacing={2}>
                            <Box>
                                <Typography variant="h5" sx={{ fontWeight: 700, color: '#D7DADC', lineHeight: 1 }}>
                                    {community.name}
                                </Typography>
                                <Typography variant="body2" sx={{ color: '#818384', mt: 0.5 }}>
                                    r/{community.name}
                                </Typography>
                            </Box>
                            {currentUserId && (
                                <Box>
                                    {isMember ? (
                                        <Button variant="outlined" onClick={() => onLeave(community._id)} sx={{ borderRadius: '999px', px: 3, textTransform: 'none', fontWeight: 700, color: '#D7DADC', borderColor: '#D7DADC', '&:hover': { borderColor: '#D7DADC', bgcolor: 'rgba(215, 218, 220, 0.05)' } }}>
                                            Joined
                                        </Button>
                                    ) : (
                                        <Button variant="contained" onClick={() => onJoin(community._id)} sx={{ borderRadius: '999px', px: 4, textTransform: 'none', fontWeight: 700, bgcolor: '#D7DADC', color: '#1A1A1B', '&:hover': { bgcolor: '#EBEDEF' } }}>
                                            Join
                                        </Button>
                                    )}
                                </Box>
                            )}
                        </Stack>
                    </Box>
                </Box>
                
                <Box sx={{ px: { xs: 1, md: 2 }, pb: 3 }}>
                    <Typography variant="body2" sx={{ color: '#D7DADC', maxWidth: '600px' }}>
                        {community.description}
                    </Typography>
                    <Stack direction="row" spacing={2} sx={{ mt: 1.5 }}>
                        <Typography variant="caption" sx={{ color: '#818384', fontWeight: 600 }}>
                            {community.members?.length || 0} Members
                        </Typography>
                        <Typography variant="caption" sx={{ color: '#818384', fontWeight: 600 }}>
                            Created by u/{community.creator?.username || 'unknown'}
                        </Typography>
                    </Stack>
                </Box>
            </Container>
        </Paper>
    );
};

export default CommunityHeader;
```

#### `frontend/src/components/communities/CommunityList.jsx`

Displays a scrollable list of communities with avatars and descriptions.

```javascript
import React from 'react';
import { Avatar, Typography, Box } from '@mui/material';
import { useNavigate } from 'react-router-dom';
import GroupsIcon from '@mui/icons-material/Groups';
import '../../styles/communityList.css';

const FALLBACK_COLORS = ["#FF4500", "#FF69B4", "#003791", "#A3AAAE", "#FF8C00", "#46D160", "#0DD3BB", "#2259FF"];

const CommunityList = ({ communities }) => {
    const navigate = useNavigate();

    return (
        <Box className="community-list-container">
            <Box className="community-list">
                {communities.map((community, index) => {
                    const displayName = community.name.startsWith("r/") ? community.name : `r/${community.name}`;
                    const avatarLetter = community.name.replace(/^r\//i, '').charAt(0).toUpperCase();
                    const bgColor = FALLBACK_COLORS[index % FALLBACK_COLORS.length];

                    return (
                        <Box
                            key={community._id}
                            className="community-list-row"
                            onClick={() => navigate(`/community/${community._id}`)}
                        >
                            <Avatar
                                src={community.avatar || undefined}
                                className="community-list-avatar"
                                sx={{ bgcolor: bgColor }}
                            >
                                {!community.avatar && avatarLetter}
                            </Avatar>
                            <Box className="community-list-info">
                                <Typography className="community-list-name">
                                    {displayName}
                                </Typography>
                                <Typography className="community-list-description">
                                    {community.description || 'No description available'}
                                </Typography>
                            </Box>
                        </Box>
                    );
                })}
                {communities.length === 0 && (
                    <Box className="community-list-empty">
                        <Typography>No communities found</Typography>
                    </Box>
                )}
            </Box>
        </Box>
    );
};

export default CommunityList;
```

#### `frontend/src/components/communities/CreateCommunityModal.jsx`

A modal dialog for creating new communities with a live preview.

```javascript
import React, { useState } from 'react';
import {
    Modal,
    Box,
    Typography,
    Button,
    Stack,
    IconButton,
    InputBase
} from '@mui/material';
import CloseIcon from '@mui/icons-material/Close';
import '../../styles/createCommunityModal.css';

const CreateCommunityModal = ({ open, handleClose, onCreate }) => {
    const [name, setName] = useState('');
    const [description, setDescription] = useState('');
    const [loading, setLoading] = useState(false);

    const handleSubmit = async (e) => {
        if (e) e.preventDefault();
        setLoading(true);
        try {
            await onCreate({ name, description });
            setName('');
            setDescription('');
            handleClose();
        } catch (error) {
            console.error("Error creating community:", error);
        } finally {
            setLoading(false);
        }
    };

    return (
        <Modal open={open} onClose={handleClose}>
            <Box className="community-modal-box">
                <IconButton onClick={handleClose} className="community-close-button">
                    <CloseIcon sx={{ fontSize: 20 }} />
                </IconButton>

                <Box className="community-header">
                    <Typography variant="h5" className="community-title">
                        Tell us about your community
                    </Typography>
                    <Typography className="community-subtitle">
                        A name and description help people understand what your community is all about.
                    </Typography>
                </Box>

                <Stack direction={{ xs: 'column', md: 'row' }} spacing={3} sx={{ mb: 4 }}>
                    {/* Left Column (Inputs) */}
                    <Box sx={{ flex: 1 }}>
                        <Box className="community-input-box">
                            <Typography className="community-input-label">
                                Community name <span>*</span>
                            </Typography>
                            <InputBase
                                fullWidth
                                placeholder=""
                                value={name}
                                onChange={(e) => {
                                    if(e.target.value.length <= 21) {
                                        setName(e.target.value);
                                    }
                                }}
                                className="community-input-base"
                            />
                            <Typography className="community-char-count">
                                {name.length}/21
                            </Typography>
                        </Box>

                        <Box className="community-input-box community-description-box">
                            <Typography className="community-input-label">
                                Description<span>*</span>
                            </Typography>
                            <InputBase
                                fullWidth
                                multiline
                                placeholder=""
                                value={description}
                                onChange={(e) => setDescription(e.target.value)}
                                className="community-description-input"
                            />
                            <Typography className="community-char-count">
                                {description.length}
                            </Typography>
                        </Box>
                    </Box>

                    {/* Right Column (Preview) */}
                    <Box sx={{ flex: 1, display: 'flex', flexDirection: 'column' }}>
                        <Box className="community-preview-card">
                            <Box className="community-preview-banner" />
                            <Box className="community-preview-content">
                                <Box className="community-preview-avatar">
                                    r/
                                </Box>
                                <Box className="community-preview-info">
                                    <Typography className="community-preview-name">
                                        r/{name || 'communityname'}
                                    </Typography>
                                    <Typography className="community-preview-stats">
                                        1 weekly visitor · 1 weekly contributor
                                    </Typography>
                                    <Typography className="community-preview-desc">
                                        {description || 'Your community description'}
                                    </Typography>
                                </Box>
                            </Box>
                        </Box>
                    </Box>
                </Stack>

                <Stack direction="row" justifyContent="flex-end" alignItems="center" className="community-footer">
                    <Stack direction="row" spacing={2}>
                        <Button onClick={handleClose} className="community-btn-back">
                            Back
                        </Button>
                        <Button onClick={handleSubmit} disabled={loading || !name} className="community-btn-create">
                            {loading ? 'Creating...' : 'Create Community'}
                        </Button>
                    </Stack>
                </Stack>
            </Box>
        </Modal>
    );
};

export default CreateCommunityModal;
```

---

## Administrative Logic

### Kick Member

The `kickMember` function removes a user from the community's member and admin lists.

**Backend Logic (`backend/services/communityService.js`):**

```javascript
const kickMember = async (communityId, adminId, targetUserId) => {
    const community = await Community.findById(communityId);
    if (!community) {
        throw createError("Community not found", 404);
    }

    // Check if the requester is an admin or creator
    const isAdmin = community.admins.some(id => id.toString() === adminId.toString());
    const isCreator = community.creator.toString() === adminId.toString();

    if (!isCreator && !isAdmin) {
        throw createError("Only the creator or admins can kick members", 403);
    }

    // Cannot kick the community creator
    if (targetUserId.toString() === community.creator.toString()) {
        throw createError("Cannot kick the community creator", 400);
    }

    // Only the creator can kick other admins
    const targetIsAdmin = community.admins.some(id => id.toString() === targetUserId.toString());
    if (targetIsAdmin && !isCreator) {
        throw createError("Only the creator can kick other admins", 403);
    }

    // Remove from both members and admins arrays
    community.members = community.members.filter(id => id.toString() !== targetUserId.toString());
    community.admins = community.admins.filter(id => id.toString() !== targetUserId.toString());
    await community.save();
    return community;
};
```

**Permission Rules:**
1. Only the **creator** or **admins** can kick members
2. The **creator cannot be kicked**
3. Only the **creator** can kick other **admins**

---

### Promote to Admin

The `promoteToAdmin` function promotes a community member to admin status.

**Backend Logic (`backend/services/communityService.js`):**

```javascript
const promoteToAdmin = async (communityId, ownerId, targetUserId) => {
    const community = await Community.findById(communityId);
    if (!community) {
        throw createError("Community not found", 404);
    }

    // Only the creator can promote members to admin
    if (community.creator.toString() !== ownerId.toString()) {
        throw createError("Only the creator can promote members to admin", 403);
    }

    // Target user must be a member
    if (!community.members.some(id => id.toString() === targetUserId.toString())) {
        throw createError("User must be a member of the community", 400);
    }

    // Already an admin - no changes needed
    if (community.admins.some(id => id.toString() === targetUserId.toString())) {
        return community;
    }

    community.admins.push(targetUserId);
    await community.save();
    return community;
};
```

**Permission Rules:**
1. Only the **creator** can promote members to admin
2. Target user **must be a member** first
3. Already promoted users are handled gracefully

---

### Permissions Matrix

| Action | Creator | Admin | Member | Non-Member |
|--------|----------|-------|--------|------------|
| Update community profile (avatar/banner) | ✅ | ✅ | ❌ | ❌ |
| Kick regular members | ✅ | ✅ | ❌ | ❌ |
| Kick other admins | ✅ | ❌ | ❌ | ❌ |
| Kick the creator | ❌ | ❌ | ❌ | ❌ |
| Promote to admin | ✅ | ❌ | ❌ | ❌ |
| Leave community | ✅ | ✅ | ✅ | ❌ |
| Join community | ✅ | ✅ | ✅ | ✅ |
| Delete community | ✅ | ❌ | ❌ | ❌ |

---

## File Summary

### Backend Files

| File | Location | Description |
|------|----------|-------------|
| `Community.js` | `backend/models/` | Mongoose schema defining community document structure |
| `communities.js` | `backend/routes/` | Express router defining all community API endpoints |
| `communityController.js` | `backend/controllers/` | HTTP request handlers that validate input and call services |
| `communityService.js` | `backend/services/` | Core business logic for all community operations |
| `auth.js` | `backend/middleware/` | JWT authentication middleware (protects routes) |
| `upload.js` | `backend/middleware/` | Multer configuration for handling image uploads |
| `cloudinaryUpload.js` | `backend/utils/` | Utility for uploading images to Cloudinary |
| `app.js` | `backend/` | Main Express app registering all routes including communities |

### Frontend Files

| File | Location | Description |
|------|----------|-------------|
| `communityService.js` | `frontend/src/services/` | API client functions for all community endpoints |
| `api.js` | `frontend/src/services/` | Axios instance with auth interceptors and base URL |
| `AppRoutes.js` | `frontend/src/routes/` | React Router configuration with community routes |
| `CommunitiesPage.js` | `frontend/src/pages/` | Page displaying all communities with join/leave |
| `CommunityDetailPage.js` | `frontend/src/pages/` | Detailed community view with posts and members tabs |
| `CommunityHeader.jsx` | `frontend/src/components/communities/` | Community banner, avatar, info, and join button |
| `CommunityList.jsx` | `frontend/src/components/communities/` | List component for displaying communities |
| `CreateCommunityModal.jsx` | `frontend/src/components/communities/` | Modal dialog for creating new communities |
| `communityList.css` | `frontend/src/styles/` | CSS styles for CommunityList component |
| `createCommunityModal.css` | `frontend/src/styles/` | CSS styles for CreateCommunityModal component |

### Related Models

| File | Location | Relationship |
|------|----------|-------------|
| `User.js` | `backend/models/` | Communities reference users in creator, members, and admins fields |
| `Post.js` | `backend/models/` | Posts reference communities via the community field |

---

## Data Flow Example

### Creating a Community

1. User fills out the `CreateCommunityModal` form on the frontend
2. Frontend calls `communityService.createCommunity({ name, description })`
3. API client sends POST request to `/api/communities` with JWT token
4. `protect` middleware validates the JWT and attaches user to `req.user`
5. `createCommunity` controller extracts data and calls service
6. `createCommunity` service:
   - Checks for duplicate names
   - Creates community with creator in members array
   - Returns the created community
7. Controller returns 201 status with community data
8. Frontend updates state and closes modal

### Admin Promotes Member

1. Creator clicks promote button on member in `CommunityDetailPage`
2. Frontend calls `communityService.promoteToAdmin(communityId, userId)`
3. API client sends POST to `/api/communities/:id/promote/:userId`
4. Controller extracts IDs and calls service
5. Service validates:
   - Requester is the creator
   - Target is a member
6. Service adds user to `admins` array
7. Returns updated community
8. Frontend refreshes community data to show updated badges