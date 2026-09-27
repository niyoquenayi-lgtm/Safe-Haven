datasource db {
  provider = "postgresql"
  url      = env("DATABASE_URL")
}

generator client {
  provider = "prisma-client-js"
}

enum Role {
  PATIENT
  SPECIALIST
  ADMIN
}

enum RiskLevel {
  LOW
  MEDIUM
  HIGH
  CRISIS
}

enum AppointmentStatus {
  CONFIRMED
  COMPLETED
  CANCELLED
}

model User {
  id                      String        @id @default(uuid())
  email                   String        @unique
  name                    String
  role                    Role          @default(PATIENT)
  createdAt               DateTime      @default(now())

  // Relationships
  assessments             Assessment[]
  patientAppointments     Appointment[] @relation("PatientRelation")
  specialistAppointments  Appointment[] @relation("SpecialistRelation")
}

model Assessment {
  id         String    @id @default(uuid())
  userId     String
  user       User      @relation(fields: [userId], references: [id], onDelete: Cascade)
  symptoms   String[]
  summary    String
  riskLevel  RiskLevel
  createdAt  DateTime  @default(now())
}

model Appointment {
  id           String            @id @default(uuid())
  patientId    String
  patient      User              @relation("PatientRelation", fields: [patientId], references: [id])
  specialistId String
  specialist   User              @relation("SpecialistRelation", fields: [specialistId], references: [id])
  scheduledAt  DateTime
  token        String            @unique @default(uuid())
  meetingUrl   String
  status       AppointmentStatus @default(CONFIRMED)
  createdAt    DateTime          @default(now())
}

import { PrismaClient } from '@prisma/client'

const globalForPrisma = globalThis as unknown as { prisma: PrismaClient }

export const prisma = globalForPrisma.prisma || new PrismaClient()

if (process.env.NODE_ENV !== 'production') globalForPrisma.prisma = prisma

import { NextResponse } from 'next/server'
import { prisma } from '@/lib/prisma'
import QRCode from 'qrcode'
import crypto from 'crypto'

export async function POST(req: Request) {
  try {
    const { patientId, specialistId, scheduledAt, meetingPlatform } = await req.json()

    if (!patientId || !specialistId || !scheduledAt) {
      return NextResponse.json({ error: 'Missing required parameters' }, { status: 400 })
    }

    // 1. Verify Specialist
    const specialist = await prisma.user.findUnique({
      where: { id: specialistId },
    })

    if (!specialist || specialist.role !== 'SPECIALIST') {
      return NextResponse.json({ error: 'Specialist not found' }, { status: 404 })
    }

    // 2. Generate Access Token & Video Link
    const token = `SHN-${crypto.randomBytes(3).toString('hex').toUpperCase()}`
    const meetingUrl = meetingPlatform === 'zoom'
      ? `https://zoom.us/j/9876543210?pwd=${token}`
      : `https://meet.google.com/lookup/${token}`

    // 3. Generate QR Code Data URI
    const qrCodeDataUrl = await QRCode.toDataURL(meetingUrl, { width: 300, margin: 2 })

    // 4. Save Appointment Record
    const appointment = await prisma.appointment.create({
      data: {
        patientId,
        specialistId,
        scheduledAt: new Date(scheduledAt),
        token,
        meetingUrl,
      },
      include: {
        specialist: true,
      },
    })

    return NextResponse.json({
      success: true,
      data: {
        id: appointment.id,
        scheduledAt: appointment.scheduledAt,
        token: appointment.token,
        meetingUrl: appointment.meetingUrl,
        specialistName: specialist.name,
        specialistEmail: specialist.email,
        qrCodeDataUrl,
      },
    })
  } catch (error) {
    return NextResponse.json({ error: 'Failed to schedule reservation' }, { status: 500 })
  }
}

'use client'

import { useState } from 'react'

interface ConfirmationData {
  id: string
  scheduledAt: string
  token: string
  meetingUrl: string
  specialistName: string
  specialistEmail: string
  qrCodeDataUrl: string
}

export default function BookingPage() {
  const [loading, setLoading] = useState(false)
  const [confirmation, setConfirmation] = useState<ConfirmationData | null>(null)

  const handleBooking = async (e: React.FormEvent<HTMLFormElement>) => {
    e.preventDefault()
    setLoading(true)

    const formData = new FormData(e.currentTarget)
    const payload = {
      patientId: formData.get('patientId'),
      specialistId: formData.get('specialistId'),
      scheduledAt: formData.get('scheduledAt'),
      meetingPlatform: formData.get('meetingPlatform'),
    }

    try {
      const res = await fetch('/api/appointments', {
        method: 'POST',
        headers: { 'Content-Type': 'application/json' },
        body: JSON.stringify(payload),
      })
      const result = await res.json()

      if (result.success) {
        setConfirmation(result.data)
      } else {
        alert(result.error || 'Booking failed')
      }
    } catch (err) {
      console.error(err)
      alert('An error occurred during booking.')
    } finally {
      setLoading(false)
    }
  }

  return (
    <div className="min-h-screen bg-slate-50 py-12 px-4 sm:px-6 lg:px-8 font-sans">
      <div className="max-w-xl mx-auto">
        <div className="text-center mb-8">
          <h1 className="text-3xl font-bold text-slate-900">Safe Haven Nexus</h1>
          <p className="text-slate-600 mt-1">Book a Confidential Consultation</p>
        </div>

        {!confirmation ? (
          <div className="bg-white p-8 rounded-2xl shadow-sm border border-slate-200">
            <form onSubmit={handleBooking} className="space-y-5">
              <div>
                <label className="block text-sm font-medium text-slate-700 mb-1">Patient ID</label>
                <input
                  type="text"
                  name="patientId"
                  required
                  placeholder="e.g. usr-patient-101"
                  className="w-full px-4 py-2.5 border border-slate-300 rounded-lg focus:ring-2 focus:ring-emerald-500 focus:outline-none"
                />
              </div>

              <div>
                <label className="block text-sm font-medium text-slate-700 mb-1">Specialist ID</label>
                <input
                  type="text"
                  name="specialistId"
                  required
                  placeholder="e.g. usr-spec-201"
                  className="w-full px-4 py-2.5 border border-slate-300 rounded-lg focus:ring-2 focus:ring-emerald-500 focus:outline-none"
                />
              </div>

              <div>
                <label className="block text-sm font-medium text-slate-700 mb-1">Preferred Date & Time</label>
                <input
                  type="datetime-local"
                  name="scheduledAt"
                  required
                  className="w-full px-4 py-2.5 border border-slate-300 rounded-lg focus:ring-2 focus:ring-emerald-500 focus:outline-none"
                />
              </div>

              <div>
                <label className="block text-sm font-medium text-slate-700 mb-1">Meeting Platform</label>
                <select
                  name="meetingPlatform"
                  className="w-full px-4 py-2.5 border border-slate-300 rounded-lg focus:ring-2 focus:ring-emerald-500 focus:outline-none bg-white"
                >
                  <option value="google">Google Meet</option>
                  <option value="zoom">Zoom</option>
                </select>
              </div>

              <button
                type="submit"
                disabled={loading}
                className="w-full bg-emerald-600 hover:bg-emerald-700 text-white font-semibold py-3 rounded-lg transition-colors shadow-sm disabled:opacity-50"
              >
                {loading ? 'Processing Reservation...' : 'Confirm Appointment'}
              </button>
            </form>
          </div>
        ) : (
          <div className="bg-white p-8 rounded-2xl shadow-sm border border-slate-200 text-center">
            <div className="inline-flex items-center justify-center w-12 h-12 bg-emerald-100 text-emerald-600 rounded-full mb-4 font-bold text-xl">
              ✓
            </div>
            <h2 className="text-2xl font-bold text-slate-900 mb-1">Reservation Confirmed</h2>
            <p className="text-slate-500 text-sm mb-6">Your access token and QR code have been generated.</p>

            {/* Specialist Details */}
            <div className="bg-slate-50 p-4 rounded-xl text-left border border-slate-100 mb-6">
              <span className="text-xs font-semibold uppercase text-slate-400 tracking-wider">Assigned Specialist</span>
              <p className="font-semibold text-slate-800 text-base mt-0.5">{confirmation.specialistName}</p>
              <p className="text-sm font-mono text-emerald-600 font-medium">{confirmation.specialistEmail}</p>
              <p className="text-xs text-slate-400 mt-2">
                Scheduled for: {new Date(confirmation.scheduledAt).toLocaleString()}
              </p>
            </div>

            {/* Token */}
            <div className="mb-6">
              <span className="text-xs font-semibold uppercase text-slate-400 tracking-wider">Session Token</span>
              <div className="text-2xl font-mono font-bold text-slate-900 bg-slate-100 py-2.5 rounded-lg border border-slate-200 mt-1">
                {confirmation.token}
              </div>
            </div>

            {/* QR Code */}
            <div className="flex flex-col items-center justify-center mb-6">
              <p className="text-xs text-slate-500 mb-3">Scan QR code to directly launch meeting</p>
              <img
                src={confirmation.qrCodeDataUrl}
                alt="Meeting QR Code"
                className="w-48 h-48 border border-slate-200 rounded-xl p-2 bg-white shadow-sm"
              />
            </div>

            {/* Direct Link */}
            <a
              href={confirmation.meetingUrl}
              target="_blank"
              rel="noopener noreferrer"
              className="inline-block w-full bg-emerald-600 hover:bg-emerald-700 text-white font-medium py-3 rounded-lg transition-colors shadow-sm"
            >
              Join Meeting Directly
            </a>
          </div>
        )}
      </div>
    </div>
  )
}

import { PrismaClient, Role, RiskLevel, AppointmentStatus } from '@prisma/client'

const prisma = new PrismaClient()

async function main() {
  console.log('🌱 Starting seed...')

  await prisma.appointment.deleteMany()
  await prisma.assessment.deleteMany()
  await prisma.user.deleteMany()

  const specialist = await prisma.user.create({
    data: {
      id: 'usr-spec-201',
      name: 'Dr. Chantal Uwitonze',
      email: 'chantal.uwitonze@safehaven.rw',
      role: Role.SPECIALIST,
    },
  })

  const patient = await prisma.user.create({
    data: {
      id: 'usr-patient-101',
      name: 'Divine Nduwimana',
      email: 'divine.nduwimana@example.com',
      role: Role.PATIENT,
    },
  })

  await prisma.assessment.create({
    data: {
      userId: patient.id,
      symptoms: ['Anxiety', 'Sleep Deprivation'],
      summary: 'Patient reports acute anxiety during exam preparations.',
      riskLevel: RiskLevel.MEDIUM,
    },
  })

  await prisma.appointment.create({
    data: {
      patientId: patient.id,
      specialistId: specialist.id,
      scheduledAt: new Date(Date.now() + 86400000),
      token: 'SHN-7F8A1B',
      meetingUrl: 'https://meet.google.com/lookup/SHN-7F8A1B',
      status: AppointmentStatus.CONFIRMED,
    },
  })

  console.log('✅ Seed finished!')
}

main()
  .catch((e) => {
    console.error(e)
    process.exit(1)
  })
  .finally(async () => {
    await prisma.$disconnect()
  })

  # 1. Install required packages
npm install @prisma/client qrcode
npm install --save-dev prisma typescript @types/node @types/react @types/qrcode tsx

# 2. Push schema to database
npx prisma db push

# 3. Seed database with mock data
npx prisma db seed
